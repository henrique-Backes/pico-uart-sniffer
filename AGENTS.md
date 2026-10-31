# AGENTS.md — pico-uart-sniffer (bridge)

Contexto operacional para agentes de código. Leia antes de qualquer alteração.

## 1. Missão

Transformar o stream textual que o firmware do Pico emite na porta serial em uma API MCP estável, com histórico consultável, log em arquivo e replay sem hardware.

Este repositório **não** contém o firmware. Ele contém:

- `spec/` — o contrato do fio (fonte da verdade)
- `host/` — a ponte em Python (CLI + MCP)
- `docs/` — decisões e pinagem

## 2. Regras invioláveis

1. **Nunca escreva em `stdout`.** O transporte MCP é stdio; qualquer `print()` corrompe o JSON-RPC. Logs vão para `stderr`.
2. **Nunca altere `spec/` para acomodar código.** Se o parser não bate com o contrato, o bug é no parser.
3. **Não invente campos** que o firmware não emite. Campo desconhecido `#...` é ignorado, não é erro.
4. **Não bloqueie o loop de leitura.** O MCP atende uma requisição por vez; `wait_for` usa timeout e o ring é acessado com lock curto.
5. **Não abra a porta duas vezes.** Uma instância por device; se estiver ocupada, retorne `port_busy` com dica.
6. **Nada de estado global mutável fora de `Link`.** Testes precisam rodar em paralelo.
7. **Toda resposta tem tamanho máximo** e devolve `cursor` + `truncated`.
8. **Trabalhe com as fixtures primeiro.** A ponte deve passar nos testes sem hardware nenhum.

## 3. Contrato (resumo)

Completo em [`spec/protocol.md`](spec/protocol.md). O essencial:

```text
#BOOT pico-uart-sniffer <ver> proto=1
#CFG baud=<n> fmt=8N1 gap=<chars> mode=text
#OVF <U1|U2> <count>
#OK <cmd>
#ERR <cmd> <code>
U1 <t_us> <hex>
U2 <t_us> <hex>
U1+ <hex>
U2+ <hex>
```

- `U1` = A→B, `U2` = B→A.
- `t_us`: 10 dígitos, µs desde o boot, 32 bits → **precisa de unwrap** no PC.
- `U1+`: continuação do frame anterior, herda o `t_us`.
- Comandos do PC: `baud <n>`, `gap <chars>`, `status`, `reset`.
- `SET_LINE_CODING` é **ignorado** pelo firmware de propósito. O baud só muda por comando.

## 4. Mapa de arquivos

| Arquivo | Responsabilidade | Não faz |
|---|---|---|
| `protocol.py` | tokenizer streaming, `Frame`, unwrap de tempo | I/O |
| `ring.py` | buffer com `seq`, cursor, evicção por bytes | parsing |
| `link.py` | abrir/fechar porta, thread de leitura, comandos | formatação |
| `log.py` | escrita, rotação, flush, sandbox de diretório | leitura |
| `replay.py` | fonte a partir de log/fixture | rede |
| `cli.py` | `ports`, `watch`, `log`, `bench`, `replay` | lógica de protocolo |
| `mcp_server.py` | tools, resources, instructions | lógica de protocolo |

`Link` e `Replay` implementam a mesma interface `Source` — o MCP não sabe qual está ativo.

## 5. Interface central

```python
@dataclass(frozen=True, slots=True)
class Frame:
    seq: int
    dir: Literal["A>B", "B>A"]
    t_us: int                  # desembrulhado, µs desde o boot
    wall: datetime             # t_us convertido para hora de parede
    data: bytes
    gap_us: int | None         # desde o frame anterior na mesma direção
    flags: tuple[str, ...]     # ex.: ("continuation", "overflow")
```

```python
class Source(Protocol):
    def read(self, since_seq: int = 0, dir: str = "both",
             max_frames: int = 50, max_bytes: int = 8192) -> ReadResult: ...
    def wait_for(self, pattern: bytes, timeout: float) -> Frame | None: ...
    def status(self) -> dict: ...
```

## 6. Comandos

```bash
cd host
uv sync
uv run pytest -q                 # testes contra spec/fixtures
uv run ruff check .
uv run uart-sniffer ports
uv run uart-sniffer watch --baud 115200
uv run uart-sniffer replay spec/fixtures/basic.txt
uv run uart-sniffer-mcp          # servidor stdio
```

## 7. Estado atual

- [x] decisões e contrato
- [x] scaffold e fixtures
- [ ] `protocol.py` + testes
- [ ] `ring.py`, `link.py`
- [ ] `log.py`, `replay.py`
- [ ] `mcp_server.py`
- [ ] `cli.py`

**Próximo passo:** implementar `protocol.py` e fazer `pytest` passar nas 6 fixtures.

## 8. Definição de pronto (P0)

- `pytest` verde em `basic`, `overflow`, `continuation`, `noise`, `baud_change`, `wrap`
- `uart_read` pagina com `cursor` e respeita `max_bytes`
- log com 1 linha por frame e rotação em 10 MB
- `replay` reproduz exatamente os `seq` e `hex` do arquivo
- nenhum `print()` fora de `stderr`

## 9. Armadilhas conhecidas

| Armadilha | Sintoma | Correção |
|---|---|---|
| `print()` no MCP | cliente fecha a conexão | usar `logging` em stderr |
| Buffer de linha não tratado | hex cortado no meio | tokenizer com estado, não `readline()` |
| `t_us` cru | gaps negativos após 71 min | unwrap acumulativo |
| `readline()` no `pyserial` | perda em rajadas | `read(4096)` + buffer próprio |
| Ring sem limite | memória cresce sem parar | evicção por bytes |
| `sleep` no `wait_for` | latência alta | `Condition` acordada no append |
| Mudar baud sem limpar estado | frames de baud antigo | `#CFG` zera cursor de gap |
