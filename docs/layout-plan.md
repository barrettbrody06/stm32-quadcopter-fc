# fc-board Rev A — PCB layout plan

Board: 50 x 50 mm, 4 layer, 1.6 mm, 30.5 x 30.5 mm M3 mounting pattern.
60 placed parts, all on the top side.

---

## 1. Board size: 50 x 50, not 36 x 36

The standard 30.5 mm FPV pattern usually sits on a 36 x 36 mm board. Do not do
that here. The part mix is 0603/1206 passives, an SOIC-8 buck, SMA and SMB
diodes (the SMB is 4.6 x 3.6 mm), a 5.0 x 3.2 mm crystal, a 4.4 x 4.2 mm
inductor, and three through-hole headers. Through-hole headers in particular
eat routing channels on every layer.

JLCPCB prices 4-layer boards in the same band anywhere under 100 x 100 mm, so
going from 36 mm to 50 mm costs nothing. Keep the 30.5 mm hole pattern centred
so it still bolts to a normal frame; the extra 7 mm of overhang on each side is
free real estate for probing during bring-up.

Shrink it on Rev B, once it works. Optimising a first board for density is
optimising the wrong thing.

## 2. Stackup — and why the layer assignment is forced

JLCPCB's standard 4-layer 1.6 mm stackup is `JLC04161H-7628`:

| Layer | | Thickness |
|---|---|---|
| L1 | 1 oz copper | 0.035 mm |
| | prepreg 7628, Dk 4.4 | **0.210 mm** |
| L2 | 0.5 oz copper | 0.0152 mm |
| | core | **1.065 mm** |
| L3 | 0.5 oz copper | 0.0152 mm |
| | prepreg 7628 | 0.210 mm |
| L4 | 1 oz copper | 0.035 mm |

The dielectric is wildly asymmetric: L1-L2 is 0.210 mm but L2-L3 is 1.065 mm,
a factor of 5. That single fact decides everything:

- **L2 = solid ground.** It sits 0.21 mm under the components, so every
  top-layer signal gets a tight, low-inductance return directly beneath it, and
  the L1/L2 pair forms real interplane capacitance.
- **L1 = every critical net.** Switching loop, crystal, USB, SPI, decoupling.
  Anything that matters goes on top where L2 is close.
- **L3 = power pours** (VBATT, +5V, +3V3). It is 1.065 mm from L2, so it is a
  poor reference plane and a poor decoupling partner — which is fine, because
  power distribution does not need either.
- **L4 = slow signals and ground fill.** Motor lines, UART, spare routing.

## 3. Ground: one plane, no splits

L2 is a single uninterrupted pour. Do not split it into "analogue" and
"digital" regions, and do not use a star ground.

Split planes and star grounds are 1990s advice for boards where the return
path was a wire. On a 4-layer board with a plane 0.21 mm away, return current
follows the signal automatically — it takes the path of least *inductance*,
which is directly under the trace. A split forces that return current to
detour around the gap, which makes a loop antenna out of the very signal you
were trying to protect.

Separate by **geometry, not by copper cuts**: keep the switcher's high-current
loop physically small so its return current stays local under it, and put the
IMU far away. That achieves what a split is trying to achieve, without the
detour.

(This corrects the "star ground" note I made earlier — it was wrong for a
4-layer board.)

## 4. Floorplan

Three zones, arranged so the noisiest and the most sensitive are at opposite
ends of the board.

```
+-------------------------------------------------------+
|  POWER   J1 - D4 - C1C2/C10 - U1 - D1 - L1 - C3-5 - U2 |   noisy
+-------------------------------------------------------+
|                                                       |
| USB-C                 [ Y1 ] [ U3 F411 ] [C20-24]     |   J5 RX
|                                                       |   J2 SWD
|                                                       |   J3 BOOT
|                      [ U4  ICM-42605 ]                |   quiet
|                                        [ J4  ESC ]    |
+-------------------------------------------------------+
```

- **Power along the top edge**, flowing left to right in schematic order so
  the current path never doubles back: battery in, protection, regulate, filter.
- **MCU in the middle.** U3 is 7 x 7 mm and lands near the board centre.
- **IMU below centre**, about 12 mm down, which puts it ~28 mm from the
  inductor — the most separation a 50 mm board allows.
- **Connectors on the edges** they naturally face: USB on the left, radio and
  debug on the right, ESC on the bottom near the arms.

### The IMU centring compromise

Ideally the gyro sits at the rotation centre, i.e. the middle of the mounting
pattern. U3 is already there, and moving the IMU to the bottom side would mean
paying for double-sided assembly. So it goes 12 mm below centre, top side.

The cost of that offset is a centripetal artifact on the **accelerometer**
(the gyro is unaffected — angular rate is the same everywhere on a rigid body):

| Rotation rate | Offset | False acceleration |
|---|---|---|
| 300 deg/s | 12 mm | 0.034 g |
| 1000 deg/s | 12 mm | 0.37 g |

At normal rates it is noise. At aggressive rates it is real, but it is also
*exactly computable*: `a_true = a_measured - w x (w x r) - alpha x r`. Measure
the offset vector `r` from the finished board and compensate in firmware.
Write the number down when you place the part.

## 5. The buck loop — the one thing most likely to sink the board

TPS54331 SOIC-8 pinout: `1 BOOT · 2 VIN · 3 EN · 4 SS · 5 VSENSE · 6 COMP ·
7 GND · 8 PH`.

Note that **VIN (2) and GND (7) are on opposite sides of the package.** That is
awkward, because the high di/dt loop is:

```
Cin(+) -> VIN(2) -> internal FET -> PH(8) -> D1 -> GND(7) -> Cin(-)
```

and it has to span the whole package width. This part is non-synchronous, so
current commutates hard between the internal FET and D1 every cycle at 570 kHz.
Loop inductance here turns directly into ringing on PH, EMI, and in bad cases
a dead FET.

Do this, in this order, before anything else on the board:

1. **C10 (10 nF) bridges pin 2 to pin 7 by the shortest possible path.** This
   is the highest-frequency part of the loop and it matters most. If the top
   side is congested, put it on L4 directly under the package with two vias.
2. **C1/C2 (4.7 uF 1206) immediately outside C10**, same loop orientation.
3. **D1 straight across pin 8 to pin 7.** Anode and cathode land in the same
   copper as the input caps' ground.
4. Only then place L1.

**The PH/switch node** is the board's biggest dV/dt aggressor. Make the copper
just big enough to carry the current and no bigger — do not pour it, do not
extend it, do not run anything under it on L1 or L4, and keep L2 unbroken
beneath it so the field terminates on the plane.

**The feedback divider R1/R2** feeds VSENSE (5), a high-impedance node. Take the
sense point at the **output capacitors**, not at the inductor, and route it
away from PH and from under L1. Keep R1/R2 close to the pin.

## 6. Other critical geometry

**Crystal (Y1, C29, C30).** Tight loop: crystal adjacent to PH0/PH1, load caps
immediately beside it, both cap grounds via'd straight down to L2 at the caps.
Nothing routed underneath on L1 or L4. A ground guard ring around the pair is
cheap and worth it. Total loop area is what matters, not trace length alone.

**VCAP1 (C27).** As close to U3 pin 22 as the footprint physically allows —
this is the internal core regulator's stability capacitor, and long traces
here cause instability, not just noise. Same idea for the three VDD 100 nF
caps: one per pin, on the pin, not shared.

**Exposed pad on U3.** Nine 0.3 mm vias in a 3x3 grid down to L2. It is a
ground connection and a thermal path.

**USB D+/D−.** Route as a pair, same layer, side by side, matched length, over
unbroken L2, no stubs, minimum vias.

Do **not** bother with controlled impedance. USB Full Speed has 4-20 ns edges;
the electrically-long threshold is ~93 mm and this board is 50 mm. The pair is
electrically short, so transmission-line behaviour never develops. Keeping the
pair together over solid ground is what actually matters, and paying JLC's
impedance-control surcharge here would buy nothing.

**SPI to the IMU.** Short, on L1, over solid ground. At 24 MHz these are the
fastest edges on the board after the switcher. Keep them clear of the power
zone entirely — route around, not through.

**Motor lines.** The 33 R series resistors go **at the STM32 end**, not at J4.
Series termination only damps reflections when it is at the driver.

## 7. Design rules (JLCPCB, no surcharges)

| Parameter | Value | Note |
|---|---|---|
| Trace / space | 0.15 mm | min is 0.1 mm; no reason to push it |
| Via | 0.3 mm drill / 0.6 mm pad | min is 0.2/0.45 |
| Annular ring | 0.15 mm min | 0.2 mm preferred |
| Power traces | 0.4 mm | ~1 A on 1 oz with margin; actual load is ~200 mA |
| Signal traces | 0.2 mm | 0.15 mm only for QFN escapes |

Set these in Board Setup *before* routing, not after.

## 8. Order of work

1. Board outline, mounting holes, edge keepouts.
2. Design rules and stackup (pick `JLC04161H-7628` in Board Setup).
3. Place **connectors first** — they are constrained by the outside world and
   everything else can move around them.
4. Place the **buck loop** per section 5. This is the least forgiving cluster.
5. Place U3, then the crystal, then decoupling on the pins.
6. Place U4 and its three caps.
7. Route in the order shown on the floorplan: power loop, crystal, VCAP1/VDD,
   IMU decoupling, USB pair, SPI, motors/UART.
8. Pour L2 (GND) and L3 (power), fill L1/L4 with ground.
9. DRC, then 3D view, then check every connector for mechanical clearance.

## 9. Before ordering

- Print the board 1:1 on paper and physically place the through-hole headers
  and the USB-C connector on it. Catches mechanical mistakes that DRC cannot.
- Check the 3D view for header/frame collisions with the 30.5 mm pattern.
- Run the JLC DFM check.
- Add test points: 3V3, 5V, VBATT, GND, PH. The PH one is optional but you will
  want it the first time the buck misbehaves.
- Order at least 5 boards. You will kill one.

## 10. Open questions before starting

- **`VBATT` hierarchical label** still exported with no consumer — delete it or
  keep it deliberately (see `parts.md`).
- **VBAT sense divider** is 10k/1k, using 35% of the ADC range. 10k/3.3k gives
  ~3x resolution. Change it before layout, not after.
- **Board outline shape** — plain 50 mm square, or corners clipped for arm
  clearance? Depends on the frame.
