# Terceiros e atribuições

## Firmware — fork de `uart_sniffer_pio`

| | |
|---|---|
| Upstream | https://github.com/jjsch-dev/uart_sniffer_pio |
| Autor | Juan Schiavoni (jjsch-dev) |
| Licença | MIT — Copyright (c) 2026 jjsch-dev |
| Commit base | *(preencher no momento do fork)* |
| Arquivos derivados | todo o repositório `pico-uart-sniffer-firmware` |
| Modificações | timestamps em µs, gap dependente de baud, protocolo textual próprio, comandos |

O texto integral da licença está em `LICENSE` no repositório de firmware, preservado do upstream.

```
MIT License
Copyright (c) 2026 jjsch-dev
```

**Modificações** — Copyright (c) 2026 pico-uart-sniffer contributors.

### O que foi mantido do upstream

- `uart_rx.pio` — programa PIO de recepção, sem alteração
- estrutura de dual-core (core0 drena, core1 formata)
- `pico_stdio_usb` / `tud_cdc_*` para o CDC

### O que foi alterado

- `ram_fifo` de `uint32_t` para `uint64_t` (byte + canal + timestamp)
- limiar de gap fixo (1000 µs) → calculado a partir do baud
- formatação de saída `[T:...]`/`[R:...]` → `U1 <t_us> <hex>`
- comandos de texto (`baud`, `gap`, `status`, `reset`)
- contador de overflow reportado por `#OVF`
- `DEFAULT_UART_BAUDRATE` 57600 → 115200
- `pico-sdk` como submódulo

---

## Dependências da ponte

| Pacote | Licença | Uso |
|---|---|---|
| [`mcp`](https://github.com/modelcontextprotocol/python-sdk) | MIT | servidor MCP (stdio) |
| [`pyserial`](https://github.com/pyserial/pyserial) | BSD-3-Clause | acesso à porta serial |

---

## Referências (sem código incorporado)

| Projeto | Licença | Influência |
|---|---|---|
| [`Noltari/pico-uart-bridge`](https://github.com/Noltari/pico-uart-bridge) | MIT | padrão de descritores CDC e baud dinâmico |
| [`qarnet/serial-mcp`](https://github.com/qarnet/serial-mcp) | MIT | formato de tools |
| [`cjznb/serialhub`](https://github.com/cjznb/serialhub) | MIT | cursor por `seq` |
| [`andrenho/poor-man-bus-pirate`](https://github.com/andrenho/poor-man-bus-pirate) | GPL-3.0 | ideia de autobaud (nenhum código usado) |

*Nenhum código GPL foi copiado, linkado ou traduzido neste projeto.*
