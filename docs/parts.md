# Parts List — STM32 Flight Controller (Rev A)

System: 3S LiPo (9.0V min, 12.6V max), USB-C bench power
5V rail load: ~200mA | 3.3V rail load: ~150mA

---

## Power

### U1 — Buck converter, 11.1V to 5V
- **Part:** TPS54331DR
- **LCSC:** C9865 (Extended)
- **Package:** SOIC-8, tape and reel
- **Why:** 3.5-28V input. 12.6V max battery x2 margin = 25.2V needed;
  28V clears it. 3A rating vs ~200mA load = heavily derated, runs cool.
  Integrated switch. Mature TI part with full app note.
- **Forces:** Non-synchronous, so REQUIRES an external Schottky catch diode.
- **Still to pick:** inductor, feedback divider, input/output caps, comp network

### D1 — TVS clamp on VBAT
- **Part:** SMBJ16A (unidirectional)
- **LCSC:** C151254 (Littelfuse, Extended)
- **Why:** 16V standoff clears 12.6V max battery with 27% margin.
  26V clamping stays under the TPS54331's 28V absolute max —
  SMBJ18A was rejected because its 29.2V clamp exceeds it.
  Unidirectional because VBAT is DC.

### D2, D3, D4 — Schottky: catch diode + power OR
- **Part:** SS34
- **LCSC:** C8678 (Basic, no setup fee)
- **Why:** 40V reverse vs 12.6V system = 3.2x margin. 3A forward
  vs ~500mA. Schottky needed for fast recovery at 570kHz switching
  and low forward drop. One part covers all three roles.

### U2 — 3.3V LDO
- **Part:** AP2112K-3.3TRG1
- **LCSC:** C51118 (Diodes Inc, Extended)
- **Why:** 6V max input gives margin over USB VBUS at 5.25V;
  the 5.5V-rated variants are too close. 250mV dropout, 600mA
  vs ~150mA load. SOT-23-5, hand-solderable.

---

## MCU
### U? — Microcontroller
- **Part:** STM32F411 (variant TBD)

## Sensors
### U? — IMU
- **Part:** ICM-42688-P

## Connectors
### J? — USB-C receptacle
### J? — SWD header