# pico-uart-sniffer

**Passive dual-channel UART sniffer for Raspberry Pi Pico, with an MCP bridge for AI coding agents.**

Sniffer UART passivo — escuta as **duas direções** de um link serial (A→B e B→A) sem nunca transmitir — e expõe os dados como **MCP server (stdio)** para OpenCode, Claude Code e afins.

> **Status:** `v0.1` em desenvolvimento. Firmware e ponte ainda não implementados.

---

## Por que

Depurar UART hoje é abrir um terminal e olhar. Um agente de código precisa de mais: saber **quem** falou, **quando**, e conseguir **buscar** no histórico sem estourar o contexto.

Este projeto entrega três coisas que um terminal não dá:

| | |
|---|---|
| **Tempo** | timestamp em µs por rajada, gerado no próprio Pico |
| **Direção** | `A>B` e `B>A` correlacionados no mesmo stream |
| **Memória** | log em arquivo, cursor por `seq`, replay sem hardware |

---

## Como funciona

```
alvo A ──TX──►──┬──► RX do alvo B
                └──► GP0  → A>B

alvo B ──TX──►──┬──► RX do alvo A
                └──► GP1  → B>A

Pico ──USB CDC (1 porta, texto)──► Ponte ──stdio MCP──► OpenCode / Claude Code
```

O Pico **nunca transmite** no link: GP0/GP1 são entradas com pull-up.

---

## Hardware

| Função | GPIO | Pino físico | Observação |
|---|---:|---:|---|
| A→B | GP0 | 1 | recebe o TX do alvo A |
| B→A | GP1 | 2 | recebe o TX do alvo B |
| GND | — | 3, 8, 13, 18, 23, 28, 33, 38 | **obrigatório** comum aos alvos |

- Alvo em **5 V** exige level shifter — o RP2040/RP2350 não é tolerante a 5 V.
- Detalhes, cuidados e casos de uso em [`docs/pinagem.md`](docs/pinagem.md).

---

## Repositórios

| Repo | Papel |
|---|---|
| **`pico-uart-sniffer`** (este) | ponte, MCP, contrato, docs |
| [`pico-uart-sniffer-firmware`](https://github.com/henrique-Backes/pico-uart-sniffer-firmware) | firmware do Pico — fork MIT do [`jjsch-dev/uart_sniffer_pio`](https://github.com/jjsch-dev/uart_sniffer_pio) |

O firmware entra como submódulo em `firmware/`.

---

## Estrutura

```
pico-uart-sniffer/
├── spec/                  # CONTRATO — fonte da verdade
│   ├── protocol.md
│   └── fixtures/          # streams de teste dos dois lados
├── host/
│   ├── pyproject.toml
│   └── src/uart_sniffer/
│       ├── protocol.py    # parser streaming
│       ├── ring.py        # seq, cursor, evicção
│       ├── link.py        # porta serial
│       ├── log.py         # arquivo + rotação
│       ├── replay.py      # fonte alternativa
│       ├── cli.py
│       └── mcp_server.py
├── docs/
├── firmware/              # submódulo
├── AGENTS.md              # contexto operacional para agentes
└── THIRD_PARTY.md
```

---

## Protocolo

Texto, uma linha por rajada, payload em hex:

```
#BOOT pico-uart-sniffer 1.0.0 proto=1
#CFG baud=115200 fmt=8N1 gap=3.5 mode=text
U1 0000001234 a501030000000ac5cd
U2 0000001567 010314022b0000
```

`U1` = A→B · `U2` = B→A · o número é o instante do primeiro byte em µs.

Contrato completo, comandos e garantias: [`spec/protocol.md`](spec/protocol.md).

---

## Começando

> Ainda não há release. Os comandos abaixo descrevem o alvo do P0.

```bash
# firmware
git clone --recurse-submodules https://github.com/henrique-Backes/pico-uart-sniffer.git
cd pico-uart-sniffer/firmware
cmake -B build -DPICO_BOARD=pico && cmake --build build -j
# copie build/*.uf2 para o Pico em BOOTSEL

# ponte
cd ../host
uv sync
uv run uart-sniffer ports
uv run uart-sniffer watch --baud 115200
```

OpenCode:

```jsonc
{
  "mcp": {
    "servers": {
      "uart": {
        "type": "local",
        "command": ["uv", "run", "--directory", "C:/caminho/pico-uart-sniffer/host", "uart-sniffer-mcp"]
      }
    }
  }
}
```

---

## Roadmap

| Fase | Entrega |
|---|---|
| **P0** | firmware com timestamps + gap; `read`, `status`, `watch`, log, replay |
| **P1** | `wait_for`, autobaud, stats, reconnect, rotação de log |
| **P2** | 921600, decoders NMEA/Modbus, export CSV/sigrok, `send` opt-in |

Critérios de aceite em [`AGENTS.md`](AGENTS.md).

---

## Documentação

| Documento | Conteúdo |
|---|---|
| [`spec/protocol.md`](spec/protocol.md) | contrato do fio (canônico) |
| [`docs/pinagem.md`](docs/pinagem.md) | ligação, níveis, ruído |
| [`docs/decisoes.md`](docs/decisoes.md) | ADRs — por que cada escolha |
| [`docs/comparativo.md`](docs/comparativo.md) | pesquisa de projetos existentes (histórico) |
| [`AGENTS.md`](AGENTS.md) | contexto para agentes de código |
| [`THIRD_PARTY.md`](THIRD_PARTY.md) | licenças e atribuições |

---

## Licença

MIT — veja [`LICENSE`](LICENSE).

O firmware deriva do [`jjsch-dev/uart_sniffer_pio`](https://github.com/jjsch-dev/uart_sniffer_pio) (MIT, © 2026 jjsch-dev). A atribuição é mantida nos arquivos e em [`THIRD_PARTY.md`](THIRD_PARTY.md).
