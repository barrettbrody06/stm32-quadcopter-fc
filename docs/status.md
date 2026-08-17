# Big Boy Drone — Status

## Board: fc-board, STM32F411CEU6 flight controller
Repo: C:\Users\Brody\Documents\KiCad\10.0\projects\stm32-quadcopter-fc
Hierarchical schematic, 5 sheets: Power, MCU, IMU, Motors, Radio

## Power system
3S LiPo (9.0–12.6V) → TPS54331 buck → 5V → AP2112K LDO → 3.3V
USB-C 5V ORed onto the 5V rail via Schottky
Battery high current does NOT go through this board — only ~200mA

## DONE
- Power sheet complete, annotated, footprints assigned, ERC clean
  (buck, LDO, TVS, Schottky OR, USB-C w/ CC resistors, VBAT sense divider)
- docs/parts.md has the full power BOM with LCSC numbers
- Custom lib at hardware/lib/fc-board — symbols/footprints pulled
  from LCSC via easyeda2kicad
- CubeMX pin assignment done, exported to docs/stm32_pinout_ref.csv

## KEY DESIGN DECISIONS
- Buck values taken from TPS54331 datasheet Table 7-1 row 1 (12V→5V)
  rather than calculated, to avoid loop stability risk
- SMBJ16A not SMBJ18A: 26V clamp stays under the buck's 28V max
- TPS54331 is non-synchronous → external SS34 catch diode required
- Input caps 50V rated (ceramic derates hard under DC bias)
- Shielded inductor because of the IMU

## NEXT: MCU sheet
- 3x VDD each need own 100nF; VDDA needs 100nF + 1uF + ferrite
- VCAP1 needs 2.2uF — internal regulator, mandatory
- 8MHz crystal on PH0/PH1, load caps from crystal datasheet
- NRST 100nF, BOOT0 10k pulldown + jumper
- SWD header: PA13 SWDIO, PA14 SWCLK, 3V3, GND
- Still to source: 2.2uF, 4.7uF 0603, 8MHz crystal, ferrite bead

## FIX OUTSTANDING
- PA1 is set GPIO_Output in CubeMX, should be GPIO_EXTI1 (IMU drives it)