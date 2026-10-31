# Pico UART Sniffer — comparativo, viabilidade e limites

> ⚠️ **Documento histórico.** Escrito antes da decisão final. Ele recomenda firmware nativo com 2 UARTs de hardware e 2 portas CDC — **ambas as escolhas foram substituídas**. As decisões vigentes estão em [`decisoes.md`](decisoes.md); o contrato, em [`../spec/protocol.md`](../spec/protocol.md). O valor deste arquivo está no levantamento de projetos e nos limites de baud, que continuam válidos.

> **Status:** documento de decisão. Nenhum firmware implementado ainda.
> **Data:** 2026-10-31
> **Alvo:** Raspberry Pi Pico (RP2040) e Pico 2 (RP2350)
> **Escopo:** sniffer passivo de 2 canais RX independentes (UART0 + UART1) → USB, com baud configurável em runtime.

---

## 1. Resposta curta

| Pergunta | Resposta |
|---|---|
| Dá para fazer? | **Sim.** O RP2040 tem 2 UARTs de hardware independentes e um controlador USB Full Speed device. Nada de externo além de fios e GND comum. |
| Dá para configurar o baud por script? | **Sim**, por três caminhos diferentes (seção 4). O mais elegante é o *line coding* do CDC: `pyserial.Serial(port, baudrate=921600)` já configura a UART no Pico. |
| Qual o baud máximo por canal (periférico)? | **7,8125 Mbaud** com `clk_peri = 125 MHz` (limite `f/16` do PL011). |
| Qual o máximo com os **2 canais ao mesmo tempo**? | **~4,5 Mbaud por canal** é o teto teórico imposto pelo USB; **3 Mbaud por canal** é o valor seguro/recomendado. |
| E com **1 canal só**? | **7,8125 Mbaud contínuos** (781 kB/s) — cabe no USB Full Speed. |
| Preciso de PIO? | Não para 2 canais. As 2 UARTs de hardware bastam e são mais simples/rápidas que PIO. PIO só faz sentido para um 3º/4º canal. |

---

## 2. Panorama GitHub

Busca feita em 2026-10-31 na API do GitHub (`pico uart sniffer`, `rp2040 uart usb bridge`, `uart sniffer rp2040`, `pico uart bridge`).

| Projeto | ★ | Licença | O que faz | O que aproveitar | Limitação para o nosso caso |
|---|---:|---|---|---|---|
| [Noltari/pico-uart-bridge](https://github.com/Noltari/pico-uart-bridge) | 609 | MIT | 2 UARTs de hardware ↔ **2 portas CDC independentes**, TX+RX, Pico e Pico 2 | Descritores USB de CDC duplo, baud por *line coding*, estrutura de projeto | É **bridge** (aciona TX). Sniffer precisa de TX desligado. Sem timestamp, sem contagem de erro. |
| [jjsch-dev/uart_sniffer_pio](https://github.com/jjsch-dev/uart_sniffer_pio) | 0 | MIT | Sniffer passivo full-duplex por PIO, **1** UART (TX+RX), RP2040/RP2350/Bus Pirate 5 | `tud_cdc_line_coding_cb` recalculando o divisor em runtime; ring buffer SPSC | 1 UART só; saída em texto hex `[T:..]/[R:..]` derruba o throughput; foco em módulos de biometria |
| [codegrande/rpico_cdc_uart](https://github.com/codegrande/rpico_cdc_uart) | 0 | verificar | Bridge PIO-UART + UART periférica, até 3,1 Mbaud, USB em núcleo dedicado | Tabela de baud medida; lições de buffer/backpressure | Bridge, 1 canal; PIO instável acima de ~3,1 Mbaud sem DMA |
| [andrenho/poor-man-bus-pirate](https://github.com/andrenho/poor-man-bus-pirate) | 7 | GPL-3.0 | Sniffer/comunicador UART/SPI/I2C com **autobaud** e config na flash | UX de configuração (`set uart.baud 921600`), autobaud | GPL-3.0 (contamina); CLI interativa não serve para streaming contínuo |
| [mdeepesh232-design/usb-logic-analyzer](https://github.com/mdeepesh232-design/usb-logic-analyzer) | — | — | LA de 6 canais com decoders UART/SPI/I2C (PIO + DMA) | Abordagem "amostra tudo e decodifica no PC" | Precisa de muito mais RAM/CPU; não é streaming contínuo |
| [gusmanb/logicanalyzer](https://github.com/gusmanb/logicanalyzer) | — | MIT | LA de 24 canais (SUMP/OLS) | Referência de captura PIO+DMA | Overkill; protocolo SUMP, não é sniffer de UART |
| [dotcypress/ula](https://github.com/dotcypress/ula) | — | MIT | µLA, LA de 16 canais a 100 MHz | Referência de performance PIO | Overkill |
| [raspberrypi/pico-examples](https://github.com/raspberrypi/pico-examples) | — | BSD-3 | `tinyusb/cdc_dual_ports`, `pio/logic_analyser` | Exemplo oficial de CDC duplo e de captura | Só exemplo |

### Conclusão da busca

**Nenhum projeto faz exatamente o recorte pedido**: 2 canais **RX-only independentes**, streaming binário contínuo, baud configurável em runtime e foco em throughput. Os dois candidatos mais próximos cobrem metades diferentes:

- **Noltari** resolve a camada USB (2 CDC independentes) + baud por *line coding*, mas é bridge e aciona TX.
- **jjsch** resolve o conceito de sniffer + baud dinâmico, mas em um único UART e com saída em texto.

Não há base que valha um fork direto: o que os dois têm de bom são **padrões** (descritores CDC duplo, callback de line coding), não código que a gente queira herdar inteiro. Um fork do Noltari economizaria ~150 linhas de descritores, mas amarraria o projeto ao escopo de bridge.

**Recomendação:** projeto próprio, MIT, citando os dois como referência.

---

## 3. Limites técnicos

### 3.1 Limite da UART (PL011 do RP2040)

O PL011 trabalha com sobreamostragem de 16×:

```
baud = f_peri / (16 × (IBRD + FBRD/64))      com IBRD ≥ 1
baud_max = f_peri / 16
```

| `clk_peri` | `baud_max` |
|---:|---:|
| 125 MHz (padrão Pico 1) | **7,8125 Mbaud** |
| 150 MHz (padrão Pico 2 / RP2350) | **9,375 Mbaud** |
| 200 MHz (overclock Pico 1) | 12,5 Mbaud |
| 250 MHz (overclock agressivo) | 15,625 Mbaud |

Observações:

- A SDK calcula `IBRD`/`FBRD` com arredondamento; o erro de quantização chega a ~0,8% nos bauds mais altos, dentro da tolerância do receptor. Some a isso o erro do transmissor.
- O piso também existe: com `IBRD = 65535`, o mínimo é ~119 baud a 125 MHz.
- Overclock **não ajuda** com 2 canais: o gargalo passa a ser o USB. Só faz sentido para 1 canal.
- A 7,8 Mbaud o bit dura 128 ns. Em fios jumper isso exige fios curtos, GND comum e alvo com clock decente. Acima de ~4 Mbaud, ruído/reflexão começa a aparecer.

### 3.2 Limite do USB Full Speed

USB 2.0 Full Speed = 12 Mbit/s, frames de 1 ms. Um endpoint bulk pode transferir no máximo **19 pacotes de 64 bytes por frame**:

```
1.216 bytes/frame × 1.000 frames/s = 1,216 MB/s  (teto absoluto, uma direção)
```

Na prática, com TinyUSB + CDC no RP2040, o número medido fica em **0,8–1,0 MB/s**. É esse número que manda no projeto.

### 3.3 Combinando UART + USB

Em 8N1 cada byte ocupa 10 bits no fio:

```
bytes/s por canal = baud / 10
bytes/s totais    = 2 × baud / 10   (2 canais simultâneos)
```

| Cenário | Baud/canal | Total | % do USB prático (~0,9 MB/s) | Veredito |
|---|---:|---:|---:|---|
| típico | 115.200 | 23 kB/s | 2,6% | trivial |
| comum | 460.800 | 92 kB/s | 10% | trivial |
| alto | 921.600 | 184 kB/s | 20% | confortável |
| alto+ | 2 Mbaud | 400 kB/s | 44% | confortável |
| **limite prático** | **3 Mbaud** | **600 kB/s** | **67%** | **recomendado como máximo** |
| agressivo | 4 Mbaud | 800 kB/s | 89% | só com `bench` validado |
| teórico | 4,5 Mbaud | 900 kB/s | 100% | sem margem alguma |
| impossível | 7,8 Mbaud | 1,56 MB/s | 173% | perda garantida |

Para **1 canal**:

| Baud | Bytes/s | % do USB prático | Veredito |
|---:|---:|---:|---|
| 3 Mbaud | 300 kB/s | 33% | folgado |
| 6 Mbaud | 600 kB/s | 67% | ok |
| **7,8125 Mbaud** | **781 kB/s** | **87%** | **viável, margem curta** |

**Rajadas:** o limite é de média, não de pico. Com ring buffer de 16 kB por canal, a 3 Mbaud dá ~27 ms de absorção por canal — rajadas curtas acima da média passam sem perda.

### 3.4 Carga de CPU (estimativa)

| Cenário | Bytes/s | Estratégia de RX | CPU estimada (125 MHz) |
|---|---:|---|---:|
| 3 Mbaud × 2 | 600 kB/s | IRQ de FIFO (gatilho 1/4) | ~15–20% |
| 4,5 Mbaud × 2 | 900 kB/s | IRQ de FIFO (gatilho 1/4) | ~25–35% |
| qualquer | — | DMA em ring + IRQ de timeout | ~8–15% |

O gargalo é o USB, não a CPU. DMA reduz a carga, mas tem uma pegadinha: o DREQ da UART só dispara no gatilho da FIFO, então os últimos 1–7 bytes de uma rajada ficam presos até chegar mais dado. A solução é combinar DMA com a IRQ de *receive timeout*.

### 3.5 PIO vs. UART de hardware

| | PL011 (UART de hardware) | PIO |
|---|---|---|
| Baud máx. teórico | f/16 → 7,8 Mbaud | f/8 → 15,6 Mbaud |
| Baud máx. medido (ref.) | não medido aqui | 3,125 Mbaud instável, 1,5625 estável ([codegrande](https://github.com/codegrande/rpico_cdc_uart)) |
| Baud não padrão | limitado pelo divisor | qualquer |
| Custo | 0 SM | 1 SM por canal |
| Formatos exóticos (9 bits, paridade) | limitado | livre |

Para 2 canais RX, a UART de hardware é a escolha certa. PIO fica para o dia em que quisermos um 3º canal ou sniffar 9 bits.

### 3.6 RP2040 vs. RP2350

| | RP2040 (Pico 1) | RP2350 (Pico 2) |
|---|---|---|
| UARTs de hardware | 2 | 2 |
| `clk_sys` padrão | 125 MHz | 150 MHz |
| Baud máx. | 7,8125 Mbaud | 9,375 Mbaud |
| RAM | 264 kB | 520 kB |
| USB | FS 12 Mbit/s | FS 12 Mbit/s |

O Pico 2 ganha em RAM (rings maiores) e um pouco em baud, mas **o USB continua sendo o teto** para 2 canais. O firmware deve compilar para os dois.

---

## 4. Como configurar o baud

### Opção A — *Line coding* do CDC (recomendada)

O host manda `SET_LINE_CODING` ao abrir a porta; o firmware intercepta e aplica na UART.

```c
// firmware: um callback por porta CDC
void tud_cdc_line_coding_cb(uint8_t itf, cdc_line_coding_t const* lc) {
    uint32_t baud = lc->bit_rate;
    if (baud >= 50 && baud <= 8000000) {
        uart_set_baudrate(itf == 0 ? uart0 : uart1, baud);
    }
}
```

```python
# host
import serial
s = serial.Serial("COM5", 921600)   # já configura a UART0 do Pico
```

| Prós | Contras |
|---|---|
| Zero protocolo; funciona com `stty`, `pyserial`, PuTTY, minicom | Depende do host enviar o line coding (pyserial e stty sempre mandam; alguns terminais só mandam ao abrir) |
| Cada porta controla seu próprio canal, naturalmente | Não persiste na flash |
| É o que o `pico-uart-bridge` faz (609★) | Não permite consultar o baud atual sem abrir a porta |

### Opção B — Comando no CDC

`baud 0 921600` em texto, ou um frame binário `[0x5A][cmd][canal][len][payload]`.

| Prós | Contras |
|---|---|
| Funciona em qualquer terminal, inclusive sem pyserial | Precisa de parser e de um modo que não colida com dados binários |
| Permite `status`, contadores de erro, modo raw/framed | Um `0x5A` perdido no meio do stream pode confundir |
| Pode salvar na flash | Mais código |

### Opção C — Configuração em compile time / `config.py`

| Prós | Contras |
|---|---|
| Trivial, zero código de protocolo | Exige recompilar/reflashar a cada mudança |
| Bom para MicroPython | Inútil para uso real de sniffer |

### Recomendação

**A + B juntos**: *line coding* como caminho principal (o script e qualquer terminal funcionam) e um comando binário mínimo para `status`, contadores de *overflow* e modo de saída. Persistir o último baud na flash é opcional (v2).

---

## 5. Arquitetura recomendada

### 5.1 Topologia USB: 2 CDC vs. 1 CDC com framing

| | **2 CDC independentes** | **1 CDC com framing** |
|---|---|---|
| Portas no PC | `COM5` (UART0) + `COM6` (UART1) | 1 porta |
| Overhead | 0% | ~2% (header de 10 B por chunk de 512 B) |
| Timestamp | só no host, resolução de ~1 ms (polling USB) | no device, ~µs (estimado) |
| Flags de erro | não | sim (overflow, troca de baud) |
| Funciona com `cat`/PuTTY | **sim** | não, precisa do script |
| Endpoints USB | 6 | 2 (+2) |
| Complexidade | menor | média |

**Recomendação para a v1: 2 CDC independentes.** É o que responde exatamente ao pedido (2 RX → USB, baud por script), funciona com qualquer terminal e tem overhead zero. O script Python lê as duas portas em threads e grava os arquivos.

**Limitação assumida:** sem timestamp no device, não dá para medir gaps inter-byte abaixo de ~1 ms. Isso é suficiente para "ver o que passa", mas **não** para analisar Modbus RTU a 115200 (3,5 caracteres = 304 µs). Se isso importar, a v2 adiciona o modo *framed*.

### 5.2 Firmware (C / pico-sdk)

```
pico-uart-sniffer/
├── firmware/
│   ├── CMakeLists.txt
│   └── src/
│       ├── main.c            # init + loop
│       ├── uart_rx.c/.h      # UART0/UART1, IRQ de FIFO + timeout, ring 16 kB
│       ├── usb_cdc.c/.h      # 2 CDC, line coding → baud
│       ├── usb_descriptors.c # descritores (padrão do pico-uart-bridge)
│       └── control.c/.h      # comandos: status, modo, contadores
└── host/
    ├── uart_sniffer.py       # CLI: ports, baud, capture, bench, raw
    └── requirements.txt      # pyserial
```

- **RX:** IRQ de FIFO (gatilho 1/4 = 8 bytes) + IRQ de *receive timeout*, copiando para ring buffer SPSC de 16 kB por canal. Sem DMA na v1 (mais simples e previsível); DMA + timeout fica como otimização.
- **Backpressure:** se o ring enche, descarta o mais antigo e incrementa `overflow`, exposto no `status`.
- **TX desligada:** os pinos TX nunca são habilitados — sniffer não pode brigar com o TX do alvo.
- **USB:** 2 funções CDC, `CFG_TUD_CDC 2`, FIFO de TX de 4 kB.

### 5.3 Script host (Python + pyserial)

```bash
python uart_sniffer.py ports
python uart_sniffer.py baud  --ch 0 921600
python uart_sniffer.py capture --ch0 921600 --ch1 115200 -o captura/
python uart_sniffer.py bench  --bytes 8000000
```

- `capture` abre as duas portas, configura o baud via line coding, lê em threads e grava `ch0.bin`, `ch1.bin` + `chunks.csv` com timestamp do host.
- `bench` mede o teto real de USB no PC do usuário (fundamental, porque 0,8–1,0 MB/s varia com o host).
- `--hex` / `--ascii` para inspeção ao vivo.

### 5.4 Pinagem

| Função | GPIO | Pino físico | Observação |
|---|---:|---:|---|
| UART0 RX | GP1 | 2 | recebe o TX do alvo 0 |
| UART0 TX | GP0 | 1 | **não usar** (sniffer é só escuta) |
| UART1 RX | GP5 | 7 | recebe o TX do alvo 1 |
| UART1 TX | GP4 | 6 | **não usar** |
| GND | — | 3, 8, 13, 18, 23, 28, 33, 38 | **obrigatório** comum com o alvo |

- Habilitar *pull-up* interno no RX evita bytes fantasma com alvo desligado.
- RP2040 **não é tolerante a 5 V**. Alvo 5 V (Arduino Uno, por exemplo) exige level shifter ou divisor resistivo.
- Alimentação pelo próprio USB do Pico.

---

## 6. Decisão recomendada

| Item | Escolha |
|---|---|
| Repositório | **Projeto próprio, MIT**, com Noltari e jjsch citados como referência |
| Firmware | **C / pico-sdk**, compilando para Pico 1 e Pico 2 |
| USB | **2 CDC independentes**, baud por *line coding* |
| Config | *line coding* + comandos `status`/`modo` |
| RX | IRQ de FIFO + timeout, ring de 16 kB por canal |
| Host | `uart_sniffer.py` (pyserial) |
| Baud alvo | **até 3 Mbaud por canal simultâneo**; 7,8125 Mbaud com 1 canal |
| Modo framed com timestamp | v2, se análise de gap for necessária |

---

## 7. Roadmap

1. **v0.1** — firmware com 1 canal (UART0), 2 CDC, line coding; validar com loopback.
2. **v0.2** — 2 canais, contadores de overflow, comando `status`.
3. **v0.3** — script `capture` + `bench`; medir o teto real no host.
4. **v1.0** — README, pinagem, casos de uso (Modbus, GPS/NMEA, console de debug), release `.uf2`.
5. **v2.0** — modo *framed* (1 CDC, timestamp no device, flags de erro) para análise de timing.
6. **v2.1** — autobaud por medição de largura de pulso; persistência na flash.

---

## 8. Como validar os números na prática

1. Grave o firmware e abra as duas portas.
2. Gere tráfego conhecido: um Pico ou adaptador USB-UART transmitindo um padrão (`0x00..0xFF`) em loop.
3. Suba o baud em degraus (921600 → 1,5M → 2M → 3M → 4M → 6M → 7,8M) e compare byte a byte.
4. Rode `bench` para medir o teto de USB do **seu** PC.
5. O ponto em que começa a faltar byte (não corromper: faltar, por overflow) é o teto real.

Referência de expectativa: a 3 Mbaud × 2, a perda deve ser zero com ring de 16 kB. A 4,5 Mbaud × 2, espere perda intermitente. A 7,8 Mbaud × 2, perda garantida.

---

## 9. Referências

- [Noltari/pico-uart-bridge](https://github.com/Noltari/pico-uart-bridge) — 2 CDC + 2 UARTs, MIT
- [jjsch-dev/uart_sniffer_pio](https://github.com/jjsch-dev/uart_sniffer_pio) — sniffer PIO com baud dinâmico, MIT
- [codegrande/rpico_cdc_uart](https://github.com/codegrande/rpico_cdc_uart) — bridge PIO-UART, tabela de baud medida
- [andrenho/poor-man-bus-pirate](https://github.com/andrenho/poor-man-bus-pirate) — autobaud e config na flash, GPL-3.0
- [raspberrypi/pico-examples](https://github.com/raspberrypi/pico-examples) — `tinyusb/cdc_dual_ports`, `pio/logic_analyser`
- [RP2040 Datasheet](https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf) — seções 4.1 (USB) e 4.2 (UART)
- [USB 2.0 Spec, §5.7.4](https://www.usb.org/document-library/usb-20-specification) — limite de 19 pacotes/frame em Full Speed
- [TinyUSB](https://github.com/hathach/tinyusb) — CDC device
