# Pinagem e ligação

## 1. Conexão

O sniffer escuta as duas linhas de um link UART existente. Ele **nunca** transmite.

```
                 ┌──────────────┐
                 │   alvo A     │
                 │  TX ─────────┼──────┐
                 │  RX ◄────────┼────┐ │
                 └──────────────┘    │ │
                                     │ │
                 ┌──────────────┐    │ │
                 │   alvo B     │    │ │
                 │  TX ─────────┼────┼─┼──┐
                 │  RX ◄────────┼────┘ │  │
                 └──────────────┘      │  │
                                       │  │
                        ┌──────────────▼──▼──┐
                        │  Pico              │
                        │  GP0 ◄── A→B (TX A)│
                        │  GP1 ◄── B→A (TX B)│
                        │  GND ── comum      │
                        └────────────────────┘
```

| Função | GPIO | Pino físico | Liga em |
|---|---:|---:|---|
| A→B | GP0 | 1 | TX do alvo A (e RX do alvo B) |
| B→A | GP1 | 2 | TX do alvo B (e RX do alvo A) |
| GND | — | 3, 8, 13, 18, 23, 28, 33, 38 | GND comum aos dois alvos |

GP0/GP1 ficam como **entrada** com pull-up interno. Não há risco de contenção: o Pico não aciona TX em nenhum momento.

## 2. Níveis elétricos

| Alvo | Ação |
|---|---|
| 3,3 V | liga direto |
| 5 V (Arduino, RS-232 TTL) | **level shifter obrigatório** — RP2040/RP2350 não são tolerantes a 5 V |
| RS-232 real (±12 V) | transceiver MAX3232 antes do Pico |
| RS-485 | transceiver MAX485 + GND, e o pino DE sempre desabilitado |

## 3. Cuidados

- **GND comum é obrigatório.** Sem ele, o sniffer lê lixo ou nada.
- **Pull-up interno** já é habilitado pelo firmware; com o alvo desligado a linha fica em nível alto e não gera bytes fantasma.
- **Fios curtos** acima de 921600. A 3 Mbaud o bit dura 333 ns; jumper de 20 cm já distorce.
- **Break longo** (linha em 0 por mais de um caractere) trava a state machine do PIO até a linha subir. Evite segurar a linha em 0.
- **Alimentação**: pelo USB do Pico. Não alimente o Pico pelo alvo.
- **Isolamento**: se o alvo tem referência flutuante (fonte chaveada ruim), use isolador digital ou optoacoplador.

## 4. Casos de uso típicos

| Cenário | Como ligar |
|---|---|
| Console de debug de um dispositivo | GP0 no TX do dispositivo, GND comum |
| Comunicação entre dois módulos | GP0 no TX do mestre, GP1 no TX do escravo |
| Modbus RTU ponto a ponto | idem, com terminador de 120 Ω no barramento |
| GPS/NMEA | GP0 no TX do GPS, baud 9600 |
| Depurar um bridge UART↔USB | GP0 e GP1 nas duas linhas do bridge |

## 5. O que não fazer

- Ligar GP0/GP1 em tensão acima de 3,3 V.
- Alimentar o Pico por 5 V no VSYS junto com o USB (backfeed).
- Deixar GND solto e usar o GND do USB como referência do alvo.
- Segurar a linha em nível baixo por muito tempo (break).
