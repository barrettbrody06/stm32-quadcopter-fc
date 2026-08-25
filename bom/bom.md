# BOM — STM32 Flight Controller (Rev A)

## Power sheet

System: 3S LiPo (9.0V min, 11.1V nominal, 12.6V max) + USB-C 5V bench power
5V rail load: ~200mA | 3.3V rail load: ~150mA
Buck design current: 1A (headroom, not actual load)

| Ref | Value / Part | LCSC | Tier | Package | Notes |
|-----|-------------|------|------|---------|-------|
| U1 | TPS54331DR | C9865 | Ext | SOIC-8 | Buck, 3.5-28V in, 3A, 570kHz, non-synchronous |
| U2 | AP2112K-3.3TRG1 | C51118 | Ext | SOT-23-5 | LDO 5V->3.3V, 6V max in, 250mV dropout |
| L1 | 6.8uH 2A shielded | C438933 | Ext | 4.4x4.2mm | 131mΩ, 2.5A sat / 2A RMS, magnetic shielded |
| D1 | SS34 | C8678 | Basic | SMA | Catch diode for U1 |
| D2 | SS34 | C8678 | Basic | SMA | OR diode, buck output -> 5V rail |
| D3 | SS34 | C8678 | Basic | SMA | OR diode, USB VBUS -> 5V rail |
| D4 | SMBJ16A | C151254 | Ext | SMB | TVS on VBAT, unidirectional |
| R1 | 10k 1% | C98220 | Ext | 0603 | Feedback high side (R_O1) |
| R2 | 1.91k 1% | C2998106 | Ext | 0603 | Feedback low side (R_O2) |
| R3 | 49.9k 1% | C23184 | Basic | 0603 | Compensation |
| R4 | 5.1k 1% | C23188 | Basic | 0603 | USB-C CC1 pulldown |
| R5 | 5.1k 1% | C23188 | Basic | 0603 | USB-C CC2 pulldown |
| C1 | 4.7uF 50V X7R | C132170 | Ext | 1206 | Input bulk |
| C2 | 4.7uF 50V X7R | C132170 | Ext | 1206 | Input bulk |
| C3 | 22uF 16V X5R | C5448976 | Ext | 1206 | Output filter |
| C4 | 22uF 16V X5R | C5448976 | Ext | 1206 | Output filter |
| C5 | 22uF 16V X5R | C5448976 | Ext | 1206 | Output filter |
| C6 | 4.7nF 50V X7R | C106218 | Ext | 0603 | Compensation (C1 in TI table) |
| C7 | 39pF 50V NP0 | C107049 | Ext | 0603 | Compensation (C2 in TI table) |
| C8 | 100nF 50V X7R | C14663 | Basic | 0603 | BOOT capacitor, mandatory |
| C9 | 10nF 50V X7R | C57112 | Basic | 0603 | Slow start, ~4ms |
| C10 | 10nF 50V X7R | C57112 | Basic | 0603 | HF input bypass |

Basic parts (no setup fee): C8678, C23184, C23188, C14663, C57112

## Design notes

**Reference:** TPS54331 datasheet Table 7-1, row 1 (12V in, 5V out, 570kHz).
Feedback and compensation values taken directly from TI's verified design
rather than calculated, to avoid loop stability risk on a first board.

**Feedback divider:** Vout = Vref x (R1/R2 + 1) = 0.8 x (10/1.91 + 1) = 4.99V
Vref is 0.772-0.828V (datasheet 6.5), so actual rail lands 4.83-5.18V.
1% resistors used so they aren't the limiting error term.

**Inductor:** 6.8uH per Table 7-1. Ripple = 973mA (Eq 9), peak = 1.49A vs
2.5A saturation (1.7x), RMS = 1.04A vs 2A rating (1.9x). Shielded because
an unshielded inductor switching at 570kHz would inject noise into the IMU.

**Catch diode required:** TPS54331 is non-synchronous (no internal low-side
switch). Without D1 the inductor's flyback voltage destroys the internal FET.

**TVS selection:** SMBJ16A not SMBJ18A. 16V standoff clears 12.6V max battery;
26V clamping stays under the TPS54331's 28V absolute max. The SMBJ18A clamps
at 29.2V, which exceeds it and defeats the protection.

**Input caps 50V rated:** ceramic capacitance collapses under DC bias. A 25V
part at 12.6V may deliver half its marked value. TI specifies 50V X7R.

**Three output caps, not two:** 3 x 22uF = 66uF nominal matches Table 7-1.
Real capacitance derates under DC bias (TI's own 94uF became ~54uF), so the
extra cap gets closer to what the compensation assumes.

**USB-C CC resistors:** 5.1k from CC1 to GND and CC2 to GND, separate
resistors. Without these a USB-C source supplies no power at all.

**Slow start:** Tss = Css(nF) x Vref / Iss = 10 x 0.8 / 2 = 4ms.
Datasheet 7.3.5 requires 1-10ms, cap must not exceed 27nF.

### Power sheet parts still missing from this BOM

Present in `power.kicad_sch` but not costed above. Values read from the
schematic; LCSC numbers are suggestions, confirm before ordering.

| Ref | Value | Suggested LCSC | Tier | Package | Notes |
|-----|-------|----------------|------|---------|-------|
| R6 | 10k 1% | C25804 | Basic | 0603 | VBAT sense divider, high side |
| R7 | 1k 1% | C21190 | Basic | 0603 | VBAT sense divider, low side |
| C11 | 1uF 50V | C15849 | Basic | 0603 | (confirm function) |
| C12 | 1uF 50V | C15849 | Basic | 0603 | (confirm function) |
| C13 | 100nF 50V X7R | C14663 | Basic | 0603 | (confirm function) |

**VBAT sense divider ratio:** 10k/1k puts 12.6 V at 1.15 V on PA0 — only 35%
of the 3.3 V ADC range. 10k/3.3k would reach 3.11 V at full charge for ~3x the
resolution, still with headroom. Source impedance either way is well under the
STM32F4 ADC limit.

## MCU sheet

STM32F411CEU6, UFQFPN-48. 3.3 V only. HSE crystal required because USB OTG FS
cannot meet USB timing off the HSI.

| Ref | Value / Part | LCSC | Tier | Package | Notes |
|-----|-------------|------|------|---------|-------|
| U3 | STM32F411CEU6 | C60420 | Ext | UFQFPN-48 | Exposed pad tied to GND |
| Y1 | X50328MSB2GI 8MHz | C115962 | Ext | SMD5032-2P | CL 20pF, ESR 80R, +/-10ppm |
| FB1 | GZ1608D601TF | C1002 | Basic | 0603 | 600R@100MHz, 200mA, 450mR — VDDA filter |
| C20 | 100nF 50V X7R | C14663 | Basic | 0603 | VDD pin 24 |
| C21 | 100nF 50V X7R | C14663 | Basic | 0603 | VDD pin 36 |
| C22 | 100nF 50V X7R | C14663 | Basic | 0603 | VDD pin 48 |
| C23 | 4.7uF 16V X5R | C19666 | Basic | 0603 | VDD bulk (AN4488 min; C19702 10uF is the typ) |
| C24 | 100nF 50V X7R | C14663 | Basic | 0603 | VBAT decoupling |
| C25 | 100nF 50V X7R | C14663 | Basic | 0603 | VDDA |
| C26 | 1uF 50V | C15849 | Basic | 0603 | VDDA |
| C27 | 4.7uF 16V X5R | C19666 | Basic | 0603 | VCAP1 — see note |
| C28 | 100nF 50V X7R | C14663 | Basic | 0603 | NRST |
| C29 | 27pF 50V C0G | C1656 | Basic | 0603 | HSE load cap |
| C30 | 27pF 50V C0G | C1656 | Basic | 0603 | HSE load cap |
| R20 | 10k 1% | C98220 | Ext | 0603 | BOOT0 pulldown (C25804 is the Basic-tier equivalent) |
| R21 | 10k 1% | C98220 | Ext | 0603 | BOOT1/PB2 pulldown |
| J2 | 1x04 header 2.54mm | — | THT | — | SWD: 1 SWDIO, 2 SWCLK, 3 +3V3, 4 GND |
| J3 | 1x02 header 2.54mm | — | THT | — | BOOT0 jumper: 1 +3V3, 2 BOOT0 |

### Design notes

**VCAP1 is 4.7 uF, not 2.2 uF.** AN4488 section on the internal regulator:
"1 x 4.7 uF low ESR < 1 ohm ceramic capacitor if only VCAP1 is provided on some
packages." The 2.2 uF figure applies to packages that expose two VCAP pins.
UFQFPN-48 has only VCAP1 (pin 22), so it takes the single 4.7 uF part.

**VBAT (pin 1) must not float.** AN4488: with no backup battery, tie VBAT to
VDD through a 100 nF decoupling cap. A floating VBAT stops the device booting.
This was missing from the original MCU checklist.

**BOOT1 (PB2) needs its own pulldown.** STM32F4 boot select uses BOOT0 *and*
BOOT1. BOOT0=0 boots main flash regardless of BOOT1, but entering the USB DFU
bootloader needs BOOT0=1 *and* BOOT1=0. PB2 resets to floating input, so
without R21 the DFU jumper on J3 is not guaranteed to work. PB2 is unused in
the pin plan, so the pulldown costs nothing.

**Crystal load capacitors — AN2867.** CL1 = CL2 = 2 x (CL - Cs).
With CL = 20 pF and an estimated stray Cs = 5 pF: 2 x (20 - 5) = 30 pF, so
27 pF is the nearest E24 value (33 pF is the other option). 27 pF gives an
effective CL of 13.5 + 5 = 18.5 pF, a few ppm fast — inside the +/-10 ppm part
tolerance. Cs = 5 pF is an estimate; measure on the real board if the frequency
matters.

**Crystal gain margin — AN2867.** gm_crit = 4 x ESR x (2*pi*F)^2 x (C0 + CL)^2.
At ESR 80 R, F 8 MHz, C0+CL 25 pF: gm_crit = 0.51 mA/V. The STM32F4 HSE
oscillator has gm_min = 5 mA/V, so gain margin = 9.9. ST requires > 5.
For comparison the 3225-package alternative (C2682774, ESR 180 R) gives a
margin of ~4.4 and was rejected.

**Ferrite on VDDA, not a plain connection.** AN4488 makes the ferrite optional,
but the ADC is reading battery voltage on a board that also carries a 570 kHz
switcher, so the filter is worth one part.

**Decoupling count.** AN4488: one 100 nF per VDD pin plus one bulk 4.7-10 uF
for the package. Three VDD pins on UFQFPN-48, hence C20-C22 plus C23.

## IMU sheet

TDK InvenSense ICM-42605, LGA-14 (2.5 x 3.0 mm), 6-axis. 4-wire SPI on SPI1.
Same register map and family pinout as the ICM-42688-P, so the driver ports
directly if the part is ever upgraded for lower noise.

| Ref | Value / Part | LCSC | Tier | Package | Notes |
|-----|-------------|------|------|---------|-------|
| U4 | ICM-42605 | C2655099 | Ext | LGA-14 2.5x3.0 | 3.8 mdps/rtHz, 24MHz SPI |
| C31 | 100nF 50V X7R | C14663 | Basic | 0603 | VDD (TDK ref circuit C1) |
| C32 | 2.2uF 16V X7R | C23630 | Basic | 0603 | VDD (TDK ref circuit C2) |
| C33 | 10nF 50V X7R | C57112 | Basic | 0603 | VDDIO (TDK ref circuit C3) |
| R22 | 10k 1% | C98220 | Ext | 0603 | AP_CS pull-up to +3V3 |

### Pin map (4-wire SPI)

| Pin | Name | Net | STM32 |
|-----|------|-----|-------|
| 1 | AP_SDO | IMU_MISO | PA6 (SPI1_MISO) |
| 4 | INT1 | IMU_INT | PA1 (EXTI1) |
| 5 | VDDIO | +3V3 | — |
| 6 | GND | GND | — |
| 7 | RESV | GND | — |
| 8 | VDD | +3V3 | — |
| 12 | AP_CS | IMU_CS | PA4 |
| 13 | AP_SCLK | IMU_SCK | PA5 (SPI1_SCK) |
| 14 | AP_SDI | IMU_MOSI | PA7 (SPI1_MOSI) |
| 2, 3, 10, 11 | RESV | no-connect | — |
| 9 | INT2 / FSYNC | no-connect | — |

### Design notes

**Decoupling is TDK's reference circuit, not a guess.** 0.1 uF + 2.2 uF on VDD,
10 nF on VDDIO. Note the 2.2 uF — not the 4.7 uF that habit suggests.

**Pin 7 RESV must go to GND.** The datasheet lists pins 2, 3, 10 and 11 as
"No Connect or Connect to GND", but pin 7 is specifically "Connect to GND".
2/3/10/11 are left as no-connects to keep escape routing sane on a 0.5 mm
pitch LGA.

**INT2/FSYNC (pin 9) is a no-connect, deliberately not grounded.** The
datasheet says ground it when FSYNC is unused, but that assumes the pin is
configured as an input. If firmware ever sets INT2 as a push-pull output and
drives it high, a hard ground is a short through the output driver. NC costs
nothing and removes that failure mode.

**10k pull-up on AP_CS.** STM32 GPIOs are high-Z during reset and before
firmware configures them, so CS floats and the IMU can latch onto noise as a
transaction. The pull-up holds it deasserted until the MCU takes over.

**VDD and VDDIO both on +3V3.** Both accept 1.71-3.6 V, and there is no
level-shifting requirement since the STM32 is a 3.3 V part.

## Motors sheet

Signal-only interface. Battery current does not pass through this board — the
ESCs tap the pack directly, so J4 carries the four DShot/PWM lines plus a
ground reference and nothing else.

| Ref | Value / Part | LCSC | Tier | Package | Notes |
|-----|-------------|------|------|---------|-------|
| J4 | 1x05 header 2.54mm | — | THT | — | 1 M1, 2 M2, 3 M3, 4 M4, 5 GND |
| R23 | 33R 1% | C23140 | Basic | 0603 | MOTOR1 series damping |
| R24 | 33R 1% | C23140 | Basic | 0603 | MOTOR2 series damping |
| R25 | 33R 1% | C23140 | Basic | 0603 | MOTOR3 series damping |
| R26 | 33R 1% | C23140 | Basic | 0603 | MOTOR4 series damping |

### Design notes

**2.54 mm header, not a JST-SH 4-in-1 connector.** A 1.0 mm 8-pin JST-SH is
what production flight controllers use and it looks tidier, but this is a first
board that has to be brought up on a bench. 0.1 in pins take a scope probe or
a logic analyser clip directly, which matters far more during bring-up than the
connector looking professional. It also works with four individual ESCs or a
4-in-1, rather than committing to one.

**33R in series on each motor line.** DShot600 has ~1.67 us bit periods and the
ESC signal wires are unterminated stubs, so edges ring. 33R at the source
damps the reflection without meaningfully slowing the edge (33R into a typical
20 pF ESC input is a sub-nanosecond time constant). Place them close to the
STM32, not close to J4 — series termination only works at the driver end.

**GND on pin 5 is a signal reference, not a return path.** Motor current
returns through the ESC's own battery leads. This pin only gives the DShot
signals a reference; keep it a thin trace and do not treat it as a power
ground.

## Radio sheet

ExpressLRS receiver over CRSF on USART1.

| Ref | Value / Part | LCSC | Tier | Package | Notes |
|-----|-------------|------|------|---------|-------|
| J5 | 1x04 header 2.54mm | — | THT | — | 1 +5V, 2 GND, 3 RX_RX, 4 RX_TX |
| C34 | 100nF 50V X7R | C14663 | Basic | 0603 | 5V decoupling at connector |
| C35 | 10uF 10V X5R | C19702 | Basic | 0603 | 5V bulk at connector |

### Design notes

**CRSF, not SBUS — and that choice is what avoids a part.** SBUS is inverted
UART, and the STM32F4 has no built-in receiver inversion (F7/H7 do), so an
SBUS-capable board needs an external inverter transistor on the RX line.
CRSF is plain non-inverted UART at 420 kbaud, so USART1 connects straight
through with no extra parts and no extra failure mode.

**Receiver runs from +5V, signals are 3.3 V.** ExpressLRS receivers take 5 V on
VCC and talk 3.3 V logic, which matches the STM32 directly. PA10 is 5 V
tolerant regardless.

**Bulk cap at the connector.** The receiver sits at the end of a cable and
draws current in bursts when its radio transmits telemetry. 10 uF + 100 nF at
J5 keeps those bursts off the shared 5 V rail rather than letting them modulate
it.

**Watch the 5 V budget.** The power sheet is specced for ~200 mA on the 5 V
rail; an ELRS receiver takes roughly 50-100 mA with peaks on transmit. It fits,
but there is not much room left for anything else on 5 V.

## Open hardware questions

- **`VBATT` is exported from the power sheet but nothing consumes it.** With
  the ESCs tapping the pack directly, no other sheet needs battery voltage.
  Either delete the `VBATT` hierarchical label on `power.kicad_sch`, or keep it
  as a deliberate export and accept one standing ERC warning.
- **C11 / C12 / C13** on the power sheet (1uF, 1uF, 100nF) are still
  undocumented and were never costed.
- **VBAT sense divider** is 10k/1k, which uses only 35% of the ADC range at
  12.6 V. 10k/3.3k would give ~3x the resolution with headroom intact.
