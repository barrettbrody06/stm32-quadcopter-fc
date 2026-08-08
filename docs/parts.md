# Power Sheet BOM — STM32 Flight Controller (Rev A)

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