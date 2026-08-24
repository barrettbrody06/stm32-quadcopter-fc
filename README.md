# STM32 Quadcopter Flight Controller

A quadcopter flight controller designed from scratch — schematic, PCB, and eventually bare-metal firmware — around an **STM32F411CEU6**. No Betaflight, no INAV, no reference design copied wholesale.

The goal is not to build a better flight controller than the ones you can buy for $30. It's to make every design decision deliberately and be able to defend each one.

---

## Status

| Stage | State |
|---|---|
| Schematic | **Complete** — 5 hierarchical sheets, ERC clean but for one deliberate export |
| PCB layout | **In progress** |
| Fabrication | Not started |
| Firmware | Not started |

*Last updated: 2026-08-21*

---

## Hardware

Board designation `fc-board`. KiCad 10.0, hierarchical schematic across five sheets.

| Sheet | Contents |
|---|---|
| **Power** | TPS54331 buck 3S → 5V, AP2112K LDO 5V → 3.3V, SMBJ16A TVS, Schottky OR with USB-C, VBAT sense divider |
| **MCU** | STM32F411CEU6, VDD decoupling, VDDA ferrite filter, VCAP1, HSE crystal, SWD header, BOOT0 jumper |
| **IMU** | ICM-42605 on SPI1, TDK reference decoupling, 10k CS pull-up |
| **Motors** | 1×05 signal header, 33R series damping on M1–M4 |
| **Radio** | 1×04 ELRS/CRSF header on USART1 |

### Design decisions worth explaining

Every non-obvious choice here has a reason, and in most cases a rejected alternative.

**The crystal was selected by calculation, not by package size.** ST's AN2867 gain-margin criterion gives the chosen X50328MSB2GI (SMD5032, 80Ω ESR) a margin of 9.9 against ST's minimum of 5. The smaller, cheaper 3225 candidate scored roughly 4.4 and was rejected — it would probably have oscillated, and "probably" across temperature and process variation is how a board works on the bench and fails in the field.

**VCAP1 is 4.7 µF, not the 2.2 µF you'll see copied around.** AN4488 specifies 4.7 µF with ESR < 1Ω for single-VCAP packages; the 2.2 µF figure belongs to dual-VCAP parts.

**VBAT is tied to +3V3 through 100nF rather than left floating.** A floating VBAT prevents the F411 from booting at all — a common first-board failure for people who reason "we aren't using the RTC backup domain."

**BOOT1/PB2 gets its own 10k pulldown.** USB DFU entry needs BOOT0 = 1 *and* BOOT1 = 0. Omit the second pulldown and the bootloader is unreachable.

**The buck compensation values come from TPS54331 datasheet Table 7-1, not from hand calculation.** A network derived by hand is a loop-stability risk requiring equipment I don't have to validate, for no benefit over TI's characterized values.

**SMBJ16A, not SMBJ18A.** The 16V part clamps at 26V, staying inside the buck's 28V absolute maximum. The 18V part would let a transient exceed it.

**Input caps are 50V rated** because ceramics derate hard under DC bias — a 25V part on a 12.6V rail loses most of its nameplate capacitance. The inductor is **shielded** specifically to protect IMU noise performance.

**The IMU is an ICM-42605, chosen over the ICM-42688-P and BMI270.** The BMI270 needs an 8 KB configuration blob uploaded over SPI on every power-up — real work in a hand-rolled driver, which this will be. The 42605 shares the ICM-426xx register map with the 42688-P, so if the better part turns out to be necessary it's a drop-in swap with no driver rewrite. That upgrade path mattered more than the spec sheet.

**INT2/FSYNC is left NC, not grounded.** The reflex is to tie unused pins to ground, but INT2 can come up configured as a push-pull output, and driving one high into a hard ground shorts the driver.

**CRSF, not SBUS.** SBUS is an inverted UART, and the STM32F411 has no RX pin inversion — the F7 and H7 do, the F4 doesn't. SBUS here would need an external inverter. CRSF is a plain 420 kbaud UART and needs nothing. This is a firmware-facing constraint that only surfaces if you read the USART chapter before choosing the connector.

**A 2.54mm motor header instead of JST-SH,** because it accepts a scope probe during bring-up and works with either four discrete ESCs or a 4-in-1.

---

## Repository layout

```
hardware/   KiCad project — schematic, PCB, footprints
firmware/   Bare-metal firmware (not started)
bom/        Bill of materials, LCSC part numbers
docs/       Design notes, calculations, decision log
test/       Test plans and bring-up procedures
```

---

## Planned firmware

Nothing here is written yet. Recording the intent so the hardware decisions above can be read against it:

- ICM-42605 sampled over SPI with DMA at 8 kHz
- DShot ESC output generated on TIM3 via DMA
- Complementary-filter attitude estimation feeding a PID rate loop
- CRSF receiver parsing on USART1

---

## Next steps

1. Finish PCB layout. Key constraints: IMU away from the switching node with its own quiet ground; USB D+/D− as a matched differential pair; crystal loop tight with a guard ring; star ground back to the input capacitor.
2. Order boards — fabrication lead time runs in the background.
3. Write firmware while waiting.
4. Keep a bring-up log: first power-on, what failed, how it was found.

---

## License

MIT — see `LICENSE`.
