# fc-board — Working Status

Internal working log. The README is the summary; this is the detail.

## Board

`fc-board` — STM32F411CEU6 flight controller. KiCad 10.0, hierarchical
schematic across 5 sheets: Power, MCU, IMU, Motors, Radio.

## Power system

3S LiPo (9.0–12.6V) → TPS54331 buck → 5V → AP2112K LDO → 3.3V
USB-C 5V ORed onto the 5V rail via Schottky
Battery high current does NOT go through this board — only ~200mA

## DONE

- **All five sheets complete**, annotated, footprints assigned.
- **ERC clean — 0 violations.** The last one was a `VBATT` hierarchical label
  exporting a net that is consumed entirely on the power sheet; demoted to a
  local label and the orphaned root-sheet pin removed.
- **No third-party library dependency.** Footprints resolve from stock KiCad
  libraries plus the project's own `hardware/lib/fc-board.pretty`, so a fresh
  clone opens without installing anything.
- `bom/bom.md` has the full BOM with LCSC numbers, per sheet, with the design
  maths behind each value.
- Custom library at `hardware/lib/fc-board` — symbols and footprints pulled from
  LCSC via easyeda2kicad.
- CubeMX pin assignment done, exported to `docs/stm32_pinout_ref.csv`.

## KEY DESIGN DECISIONS

- Buck values taken from TPS54331 datasheet Table 7-1 row 1 (12V→5V) rather than
  calculated, to avoid loop stability risk on a first board
- SMBJ16A not SMBJ18A: 26V clamp stays under the buck's 28V max
- TPS54331 is non-synchronous → external SS34 catch diode required
- Input caps 50V rated (ceramic derates hard under DC bias)
- Shielded inductor because of the IMU
- **VCAP1 = 4.7 µF, not 2.2 µF.** AN4488 specifies a single 4.7 µF, ESR < 1Ω for
  packages exposing only VCAP1. UFQFPN-48 is one of those. The 2.2 µF figure
  belongs to dual-VCAP packages and is the value most copied designs use.
- Crystal X50328MSB2GI selected by AN2867 gain margin (9.9 vs ST's minimum of
  5); the smaller 3225 candidate scored ~4.4 and was rejected
- BOOT1/PB2 gets its own 10k pulldown — DFU needs BOOT0=1 *and* BOOT1=0
- CRSF not SBUS: SBUS is inverted and the F411 has no RX pin inversion

## NEXT: PCB layout

- IMU away from the switching node, with its own quiet ground
- USB D+/D− routed as a matched differential pair
- Crystal loop tight, with a guard ring
- Star ground back to the input capacitor
- Order boards early — fabrication lead time runs in the background

## OUTSTANDING

- **PA1 is set GPIO_Output in CubeMX, should be GPIO_EXTI1** (the IMU drives it).
  Not yet corrected; no `.ioc` exists in the repo, so this lives only in notes
  until firmware work starts.
- VBAT sense divider is 10k/1k, putting 12.6 V at 1.15 V — 35% of the 3.3 V ADC
  range. 10k/3.3k reaches 3.11 V for roughly 3× the resolution, still with
  headroom.
- C11/C12/C13 on the power sheet (1 µF, 1 µF, 100 nF) are undocumented and
  uncosted — confirm their function.
- Y1 tolerance: BOM says ±10 ppm, the LCSC listing says ±20 ppm. Either meets
  USB full-speed timing comfortably, but reconcile before ordering.
- 15 symbol `lib_id`s still name SparkFun libraries not in `sym-lib-table`.
  Cosmetic — symbol definitions are cached in the schematics, so a clean clone
  renders and ERCs correctly. **Do not "fix" this by remapping to `Device:R` /
  `Device:C`:** SparkFun's parts are 0.2" pin pitch against KiCad's 0.3", so the
  pins leave the wire ends and ERC explodes. The correct route is copying the
  SparkFun symbols into `fc-board.kicad_sym` so geometry is identical, then
  repointing to `fc-board:*`.
