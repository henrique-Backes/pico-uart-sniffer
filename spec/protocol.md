# Contrato do fio — `proto=1`

> **Status:** congelado para a v0.1
> **Fonte da verdade:** este arquivo. O firmware e a ponte o implementam; nenhum dos dois o redefine.
> **Transporte:** USB CDC (ACM), 1 porta, texto UTF-8/ASCII, linhas terminadas em `\r\n`.

---

## 1. Direção firmware → PC

### 1.1 Linhas de controle

```
#BOOT pico-uart-sniffer <versão> proto=1
#CFG baud=<n> fmt=8N1 gap=<chars> mode=text
#OVF <U1|U2> <total>
#OK <comando>
#ERR <comando> <código>
```

| Campo | Regra |
|---|---|
| `<versão>` | semver, ex.: `0.1.0` |
| `baud` | taxa do link alvo, inteiro |
| `gap` | multiplicador em tempos de caractere (float, ex.: `3.5`) |
| `<total>` | contador **cumulativo** de bytes descartados no canal |
| `<código>` | `bad_args`, `range`, `unsupported`, `busy` |

`#BOOT` é emitido **uma vez por conexão**, logo após o host abrir a porta, e sempre antes de qualquer `U1`/`U2`. Em reconexão, é emitido de novo.

`#CFG` é emitido depois do `#BOOT` e a cada mudança de configuração.

### 1.2 Linhas de dados

```
U1 <t_us> <hex>
U2 <t_us> <hex>
U1+ <hex>
U2+ <hex>
```

| Campo | Regra |
|---|---|
| tag | `U1` = direção A→B · `U2` = direção B→A |
| `t_us` | **10 dígitos**, zero-padded, µs desde o boot, módulo 2³² |
| `hex` | minúsculo, contíguo, sem separador; `""` é proibido |
| `+` | continuação do frame anterior **da mesma tag** |
| tamanho | ≤ **256 bytes** por linha (512 chars hex) |

Regras de montagem:

1. Um frame é uma rajada delimitada por silêncio (seção 3).
2. `t_us` é o instante do **primeiro byte** da rajada.
3. Se a rajada passar de 256 bytes, ela é dividida: primeira linha com `t_us`, seguintes com `+` e **sem timestamp**.
4. Continuações preservam a ordem; o receptor concatena até a próxima linha sem `+`.
5. Uma linha `U1+` nunca aparece sem um `U1` anterior.
6. As continuações são emitidas **imediatamente** após a base: nenhuma linha se intercala entre `U1` e os `U1+` seguintes.

### 1.3 Linhas desconhecidas

Qualquer linha que comece com `#` e não seja reconhecida deve ser **ignorada** pelo receptor (forward compatibility). Qualquer linha que não comece com `#`, `U1` ou `U2` também é ignorada e contada como `line_errors`.

---

## 2. Direção PC → firmware

Uma linha por comando, terminada em `\n`. Case-sensitive, minúsculo.

| Comando | Efeito | Resposta |
|---|---|---|
| `baud <n>` | muda a taxa do link alvo, `50 ≤ n ≤ 8000000` | `#OK baud` + novo `#CFG` |
| `gap <chars>` | multiplicador de gap, `0.5 ≤ chars ≤ 100` | `#OK gap` + `#CFG` |
| `status` | pede estado | `#CFG` com a config atual |
| `reset` | limpa ring e contadores, mantém baud | `#OK reset` |
| `mode text` | modo texto (único suportado na v1) | `#OK mode` |
| `mode raw` | reservado | `#ERR mode unsupported` |

Erros:

| Situação | Resposta |
|---|---|
| argumento não numérico | `#ERR <cmd> bad_args` |
| fora da faixa | `#ERR <cmd> range` |
| não implementado | `#ERR <cmd> unsupported` |
| comando durante mudança de baud | `#ERR <cmd> busy` |

### 2.1 Baud

- `SET_LINE_CODING` do CDC é **ignorado**. Abrir a porta com qualquer baudrate não altera o link alvo.
- Ao aplicar `baud`: descarta os bytes pendentes, zera contadores e emite `#CFG`.
- Baud inicial após o boot: **115200**.

---

## 3. Segmentação

Uma rajada termina quando ocorrer **qualquer** uma das condições:

| Condição | Valor |
|---|---|
| silêncio | `gap × 10 / baud` segundos (default `3.5` caracteres) |
| tamanho | 256 bytes acumulados |
| latência | 5 ms desde o primeiro byte pendente |

`gap` em tempos de caractere, considerando 8N1 (10 bits por caractere).

| Baud | 3,5 caracteres |
|---:|---:|
| 9600 | 3,65 ms |
| 57600 | 608 µs |
| 115200 | 304 µs |
| 921600 | 38 µs |

---

## 4. Tempo

| Item | Regra |
|---|---|
| Base | µs desde o boot do Pico (`time_us_32`) |
| Emissão | 10 dígitos, módulo 2³² |
| Wrap | a cada ~71,58 min |
| Precisão | ±2 tempos de byte (amostragem por drenagem) |
| Reconstrução | o receptor acumula deltas de 32 bits, nunca compara valores absolutos |

Algoritmo do receptor:

```text
delta = (t_us_novo - t_us_anterior) & 0xFFFFFFFF
absoluto += delta
```

O primeiro frame de uma conexão define a origem. `wall = t0_wall + (absoluto - absoluto_0)/1e6`.

---

## 5. Garantias

| Garantia | Valor |
|---|---|
| Ordem dentro de uma direção | **exata** |
| Ordem entre direções | aproximada à ordem de drenagem (µs) |
| Timestamp | ±2 tempos de byte |
| Fronteira de rajada | gap declarado, com erro < 1 caractere |
| Perda | **reportada** por `#OVF`, nunca silenciosa |
| Erro de framing/paridade | **não detectado** na v1 (limitação do PIO) |

---

## 6. Limitações conhecidas da v1

| Limitação | Efeito | Contorno |
|---|---|---|
| PIO em 8× | teto prático ~1,5–3 Mbaud | usar ≤ 921600 |
| Sem detecção de framing error | só `#OVF` | verificar no alvo |
| Só 8N1 | sem 7E1/paridade | v2 |
| Break longo (> 1 caractere em nível baixo) | SM espera a linha subir e perde bytes | reduzir ruído / pull-up |
| Um link | 2 fios | 2 Picos |

---

## 7. Exemplo completo

```text
#BOOT pico-uart-sniffer 0.1.0 proto=1
#CFG baud=115200 fmt=8N1 gap=3.5 mode=text
U1 0000001234 a501030000000ac5cd
U2 0000001567 010314022b0000
U1 0000002001 a501030000000ac5cd
U2 0000002030 0103020050b9
#OVF U1 3
U1 0000003000 48656c6c6f
```

Leitura:

- `A>B` enviou 9 bytes em `t=1234 µs`; `B>A` respondeu 7 bytes em `t=1567 µs`.
- `#OVF U1 3` significa **3 bytes perdidos no total** no canal A→B até agora.
- `48656c6c6f` = `Hello`.

---

## 8. Fixtures

`spec/fixtures/*.txt` são streams que ambos os lados devem reproduzir/parsear:

| Arquivo | Cobre |
|---|---|
| `basic.txt` | frames simples nas duas direções |
| `overflow.txt` | `#OVF` no meio do stream |
| `continuation.txt` | rajada > 256 B com `+` |
| `noise.txt` | linha truncada, lixo, `#` desconhecido |
| `baud_change.txt` | `#CFG` + `#OK` durante a captura |
| `wrap.txt` | virada de 2³² em `t_us` |

---

## 9. Histórico

| Versão | Mudança |
|---|---|
| 1 | contrato inicial: texto + hex, timestamps em µs, segmentação por gap |
