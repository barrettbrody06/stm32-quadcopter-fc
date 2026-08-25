# Datasheets

Links rather than copies. Every document below belongs to its manufacturer, and
mirroring vendor PDFs in a repo under an MIT license muddies what the license
actually covers. Links also stay current — several of these have been revised
since the board was designed.

LCSC part numbers match `bom/bom.md`. Where a part has no first-party
manufacturer page, the LCSC product page is the authority, since that is where
the part is sourced.

## Active devices

| Ref | Part | Manufacturer | Datasheet | LCSC |
|---|---|---|---|---|
| U1 | TPS54331DR | Texas Instruments | [ti.com/lit/gpn/TPS54331](https://www.ti.com/lit/gpn/TPS54331) | [C9865](https://www.lcsc.com/product-detail/C9865.html) |
| U2 | AP2112K-3.3TRG1 | Diodes Incorporated | [diodes.com/datasheet/download/AP2112.pdf](https://www.diodes.com/datasheet/download/AP2112.pdf) | [C51118](https://www.lcsc.com/product-detail/C51118.html) |
| U3 | STM32F411CEU6 | STMicroelectronics | [st.com/resource/en/datasheet/stm32f411ce.pdf](https://www.st.com/resource/en/datasheet/stm32f411ce.pdf) | [C60420](https://www.lcsc.com/product-detail/C60420.html) |
| U4 | ICM-42605 | TDK InvenSense | [DS-000292](https://www.invensense.tdk.com/en-us/download-resource/ds-000292-icm-42605-datasheet) | [C2655099](https://www.lcsc.com/product-detail/C2655099.html) |

## Passives and discretes

| Ref | Part | Manufacturer | Datasheet | LCSC |
|---|---|---|---|---|
| Y1 | X50328MSB2GI, 8 MHz | YXC | [C115962](https://www.lcsc.com/product-detail/C115962.html) | [C115962](https://www.lcsc.com/product-detail/C115962.html) |
| L1 | CMLO0420H6R8MTT, 6.8 µH | Cybermax | [C438933](https://www.lcsc.com/product-detail/C438933.html) | [C438933](https://www.lcsc.com/product-detail/C438933.html) |
| D1–D3 | SS34 | MDD (Microdiode) | [C8678](https://www.lcsc.com/product-detail/C8678.html) | [C8678](https://www.lcsc.com/product-detail/C8678.html) |
| D4 | SMBJ16A | Littelfuse | [C151254](https://www.lcsc.com/product-detail/C151254.html) | [C151254](https://www.lcsc.com/product-detail/C151254.html) |
| USBC1 | TYPE-C-31-M-12 | — | [LCSC search](https://www.lcsc.com/search?q=TYPE-C-31-M-12) | not recorded — confirm before ordering |

## Application notes the design leans on

These are cited by name in the design decisions and are worth reading alongside
the schematic.

| Doc | Subject | Where it is used |
|---|---|---|
| [AN2867](https://www.st.com/resource/en/application_note/an2867-guidelines-for-oscillator-design-on-stm8afals-and-stm32-mcusmpus-stmicroelectronics.pdf) | Oscillator design for STM32 | Gain-margin calculation that selected Y1 over a 3225 candidate |
| [AN4488](https://www.st.com/resource/en/application_note/an4488-getting-started-with-stm32f4xxxx-mcu-hardware-development-stmicroelectronics.pdf) | STM32F4xxxx hardware development | VCAP1 = 4.7 µF, VDD decoupling, VBAT and BOOT pin handling |
| TPS54331 datasheet, Table 7-1 | Verified buck design values | Feedback divider, compensation network, inductor and output caps |

## Note on Y1

`bom/bom.md` records the crystal as ±10 ppm; the LCSC listing for C115962 states
±20 ppm. Either satisfies USB full-speed timing by a wide margin, but the BOM
figure should be reconciled against the vendor page before an order goes out.
