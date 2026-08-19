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
