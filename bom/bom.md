# Bill of Materials — Confirmed Buy List

Full context, specs, and compatibility notes live in the Computer Engineering Projects Tracker (Total Life Organization project, Perplexity). This is the buy-list snapshot as of the hardware finalization on Jul 31, 2026.

| # | Item | Link | Price |
|---|---|---|---|
| 1 | STM32 Nucleo-F411RE | https://www.amazon.com/dp/B07JYF8RRB | ~$20-26 |
| 2 | GY-91 10DOF IMU (MPU-9250 + BMP280) | https://www.amazon.com/gp/product/B0FKMY17VR | $9.99 |
| 3 | FlySky FS-iA6B receiver (iBus) | https://www.amazon.com/gp/product/B0CN2XFKBT | $16.98 |
| 4 | HiLetgo MCP2515 CAN module 2-pack | https://www.amazon.com/gp/product/B01D0WSEWU | $10.59 |
| 5 | ZOWZEA M2 anti-vibration rubber balls, 24pc | https://www.amazon.com/gp/product/B0G4M881V9 | $6.99 |

Total: ~$64.55-$70.55

## Already owned (not repurchased)

- Power Distribution Board (PDB) — QWinOut PDB XT60 BEC 5V & 12V
- ESCs x4 + motors x4 + propellers — Goolsky A2212 1000KV combo kit
- FlySky FS-i6 transmitter (paired with the new FS-iA6B receiver)
- LiPo battery + charger
- Raspberry Pi
- Ender V3 3D printer
- ECE110/220 kit components
- Solder station

## Key firmware-relevant compatibility notes

- ESC PWM moved to TIM3 (PB4/PB5/PB0/PB1) to avoid a pin conflict with USART1_RX (PA10), which the FS-iA6B iBus line uses.
- The HiLetgo MCP2515 module is 5V-only on both chips (shared VCC). STM32 side is 5V-tolerant (safe, but marginal without a fix); the Raspberry Pi side is NOT 5V-tolerant and needs a split-VCC trace-cut mod or a logic-level shifter before connecting.
- FS-iA6B iBus output is native 3.3V logic — safe into PA10 with no modification.

See the full tracker for pin maps, wiring details, and sourcing citations.
