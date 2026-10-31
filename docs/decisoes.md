# Decisões de projeto (ADR)

Registro curto do **porquê**. Cada decisão lista a alternativa descartada — se ela voltar à mesa, comece por aqui.

| # | Decisão | Descartado | Motivo |
|---|---|---|---|
| **D1** | Firmware = **fork do `uart_sniffer_pio`** (PIO, 2 canais) | firmware nativo com 2 UARTs de hardware | reusa PIO, descritores USB e o dual-core; ~80 linhas de patch em vez de ~520 |
| **D2** | Tap **passivo**, TX nunca habilitado | bridge bidirecional | evita contenção elétrica com o alvo |
| **D3** | **1 CDC** com protocolo textual | 2 CDC raw (2 portas COM) | direção e tempo correlacionados; 1 porta; sem risco de dessincronizar baud |
| **D4** | Payload em **hex** | texto cru | binário, `\n` no payload e colisão de prefixo quebram o parser |
| **D5** | Timestamp e gap gerados **no firmware** | timestamp no PC | o PC só vê lotes de ~1 ms do USB |
| **D6** | Segmentação por gap no firmware | segmentação no PC | o PC não vê os bytes individuais chegando |
| **D7** | Baud só por **comando textual**; `SET_LINE_CODING` ignorado | line coding do CDC | abrir a porta não pode sobrescrever o baud configurado |
| **D8** | MCP **stdio** | daemon HTTP | um harness por vez, sem concorrência |
| **D9** | Log no **PC** | flash do Pico | desgaste da flash, RAM e acesso ao arquivo |
| **D10** | Licença **MIT** | GPL-3.0 | compatível com o fork; reversível |
| **D11** | `pico-sdk` como **submódulo** | `$ENV{PICO_SDK_PATH}` | reprodutibilidade do build |
| **D12** | **Python + SDK oficial MCP** | Rust/rmcp | iteração rápida, `pyserial`, ecossistema |
| **D13** | **Dois repos** + submódulo | monorepo | PR upstream limpo e paralelismo real |
| **D14** | Nomes **Esquema A** (`pico-uart-sniffer` + `-firmware`) | org / simetria | descritivo, livre no GitHub e no PyPI, escala |

---

## D1 — por que fork e não firmware próprio

A primeira versão do plano previa firmware nativo com as duas UARTs de hardware. A leitura do código do jjsch mostrou que:

| Parte | Estado | Aproveitável? |
|---|---|---|
| `uart_rx.pio` | 8× oversampling, pull-up, 20 instruções | sim, intocado |
| core0 (drena PIO → FIFO) | correto | sim, com timestamp |
| `ram_fifo` | SPSC sem lock | sim, muda para u64 |
| core1 (formatação) | **1 ms fixo, sem timestamp** | não, reescrever |
| USB CDC | `pico_stdio_usb` + `tud_cdc_write` | sim |

O que está errado é justamente a parte pequena. Reescrever o resto custaria ~520 linhas; o patch custa ~80.

**Custo assumido:** teto de ~1,5–3 Mbaud (PIO) em vez de 7,8 Mbaud (UART de hardware) e 8N1 fixo. Irrelevante para 9600/115200.

---

## D7 — por que ignorar o line coding

O `pyserial` envia `SET_LINE_CODING` **ao abrir** a porta. Se o firmware honrasse isso, abrir em `115200` sobrescreveria um `baud 9600` configurado antes — corrida silenciosa.

Com o comando textual, a ordem é explícita: abre, pergunta (`status`), configura (`baud`).

---

## D3 — por que 1 CDC e não 2

Com 2 portas, o PC não sabe qual byte veio primeiro entre as duas direções, e o baud de cada porta é uma armadilha (abrir a porta errada reconfigura o canal errado). Com 1 porta e framing, a ordem e o tempo vêm do device.

Custo: ~2% de overhead e texto em vez de binário puro — aceitável até ~921600.

---

## D14 — por que os nomes

Dois repos no mesmo dono não podem ter o mesmo nome. O tronco `pico-uart-sniffer` identifica o projeto; o sufixo `-firmware` diz o papel. Se um dia virar organização, o tronco sobrevive.
