# Board A audit — 2026-10-08 (pre-order)

**Scope:** `Espressomaschine Board A.kicad_sch` (20:11) and `.kicad_pcb` (saved 20:21, 4-layer, 1.6 mm,
100.2 × 76.5 mm, 111 footprints, KiCad 10). Checked against `BOARD-A.md`, `espresso.gaphor` (HN-1…HN-8,
REQ-SYS/REQ-SUB), the JLCPCB Tools project DB and the LCSC/JLC part data.

**Method:**
- KiCad ERC and DRC (all severities, schematic parity, zones refilled).
- Netlist and part-rating review.
- Fabrication rules against JLC 4-layer capabilities.
- Plugin DB vs board consistency.
- Model and spec trace.

No design files were changed.

Previous audit and re-checks: `Board-A-Audit-2026-09-24.md`. Every open item from it is now either fixed or
closed by a user decision (see the end of this file).

## Scoring

Same scale as the previous audit:
- **U** (urgency) 1–5: 5 = board dead or guaranteed respin · 4 = wrong order or function at risk · 3 = robustness
  or spec deviation · 2 = bring-up, documentation, traceability · 1 = cosmetic or note.
- **E** (effort) 1–5: 1 ≤ 15 min · 2 ≤ 1 h · 3 ≤ half a day · 4 ≤ 1 day · 5 = needs an external decision.
- **P = U / E**, ties broken by higher U.

## Ranked findings

| # | P | U | E | Finding & evidence | Ref |
|---|---|---|---|---|---|
| 1 | 4.0 | 4 | 1 | **Fabrication outputs are stale.** `jlcpcb/gerber/*` and `production_files/BOM/CPL` are from 19:44, but the PCB was saved again at 20:21 and the J10 fields changed. Regenerate them with JLCPCB Tools immediately before uploading. Never order from the old set. (They are git-ignored, so nothing in the repo shows which set was ordered.) | Fab |
| 2 | 4.0 | 4 | 1 | **Check the relay NO/NC mapping by continuity before soldering K1/K2.** The W11 footprint maps symbol pad 1 (NO) → datasheet pin 12 and pad 2 (NC) → pin 1. This was read from TE 108-98010 p.5 (A version, symmetrical coil). With the relay unpowered, COM (pin 6/7) ↔ pin 1 must beep. If the mapping were swapped, the valve signal would be active at rest and violate REQ-SYS-074 / REQ-SUB-PROT-015 (valve rests in tray path). | Blk 8, REQ-SYS-074 |
| 3 | 2.0 | 2 | 1 | **F2 hold-current derating.** SMD1812P075TF/24 holds 0.75 A at 25 °C. The worst-case 12 V load is about 0.58 A: HMI ≤ 0.5 A, 2 × 37.5 mA relay coils and the D2 LED. PPTC hold current drops to roughly 70–75 % at 60 °C (≈ 0.53–0.56 A), so a warm machine with the HMI at full load can nuisance-trip. Options: keep it (if the HMI really stays well below 0.5 A), or use SMD1812P110TF/24 (C20999, same 1812 footprint). If you change it, F2 > 1 A exceeds the single-pin FFC contact rating, so put +12 V on two J8 pins. | Blk 2, J8 |
| 4 | 2.0 | 2 | 1 | **Identical connectors for power and sensors.** J1 (+5 V **in**) and J11/J12/J14 (FTH NTC, tank NTC, level switch) are all JST XH 2-pin, so a supply plug fits a sensor socket. 5 V into J11/J12 is current-limited by the 1 kΩ series resistor (about 1.4 mA into the clamp), so this is unlikely to kill anything but gives silent wrong readings. Mitigate with clear silkscreen labels at each connector and/or a different housing colour or keyed cable set. | Blk 2/6 |
| 5 | 2.0 | 2 | 1 | **Spec errata in `BOARD-A.md`** (the schematic is right in every case): (a) Block 13 table says D+ → GPIO19 / D− → GPIO20; on the S3, **GPIO19 = D−, GPIO20 = D+**. (b) Block 3 still describes MCP9600 breakouts on female headers; the board uses bare MCP9600-E/MX plus WAGO 2060 terminals. (c) Block 7 still describes a bare HX711 (SCK/DOUT + 100 Ω); it is an I²C module on J7. (d) Block 12 still lists HX711 SCK/DOUT. (e) Connector summary lists J5/J6 as tank NTC/level and J9 as 11 pins; as built, J5/J6 are TEMP0/TEMP1 and J9 is 12 pins with 2 GND. (f) Block 8 says "second relay footprint DNP"; K2 is populated. | traceability |
| 6 | 2.0 | 2 | 1 | **Model not updated for the I²C HX711 module.** MS-3/MS-4 and REQ-SUB-MAS-004 ("keep the converter clock line low…") still describe a bare HX711 driven by SCK/DOUT, which no longer exists on HN-1. Re-word it as a module requirement (crystal, 10 SPS, I²C address ∉ {0x60, 0x67}) so it stays verifiable. | model |
| 7 | 1.5 | 3 | 2 | **No mounting holes.** The board has no MountingHole or NPTH fixing footprints. It sits in a machine with a vibrating ULKA pump (HN-8 notes 50 Hz vibration), and the hand-soldered relays, electrolytics and connectors would carry the board on their joints. Add 3–4 M3 holes (pad to GND, or isolated) on the board outline, or confirm another retention method (card guide, clamp). | mech |
| 8 | 1.5 | 3 | 2 | **Off-board I²C lines are unprotected.** SDA/SCL go straight from J3 (pressure sensor) and J7 (remote HX711 module at the load cell) to IO8/IO9 and both MCP9600s, with no series resistance or ESD clamp. These are the longest cables on the board and run past pump and heater wiring. A transient on them hits the ESP32 and both temperature converters at once. Cheap fix: 33–100 Ω series per line at J3/J7 and/or a 2-line ESD array (e.g. the USBLC6 type already in the BOM). Also confirm the pull-ups fitted on the pressure sensor and HX711 modules, since they add in parallel to R4/R5 (4k7). | Blk 4/7, REQ-SUB-SEN-002 |
| 9 | 1.0 | 1 | 1 | **ERC:** 3 × `power_pin_not_driven` (U6 VIN on +5V, J13 VSS on GND, #PWR06 on +12V). Add PWR_FLAGs. The GND/EXP name warning is harmless. | ERC |
| 10 | 1.0 | 1 | 1 | **DRC leftovers:** two 0.006 mm dangling stubs on the IO40 track at (188.72/188.98, 49.78), and the 'V02' silkscreen text stroke is too thin. Excluded and acceptable: J13 pad 10 at 0.435 mm from the edge (unused card-detect pad, above JLC's 0.3 mm), U5 silkscreen clipped at the edge, U5 differs from the library footprint (thermal vias). | DRC |
| 11 | 1.0 | 1 | 1 | **Firmware constraint to document:** CNTCTR_AUX is on RXD0 (IO44). The ESP-IDF console must run on USB-Serial-JTAG, and UART0 RX must not be claimed. The ROM boot log on TXD0 is harmless (TXD0 is not connected). | Blk 9 |

### Critical path before ordering
1. Close #2: measure one relay with a multimeter (5 min).
2. Decide #7: mounting holes. This is the only remaining layout change.
3. Optionally fix #8 and #3 in the same layout pass.
4. Regenerate Gerbers, BOM and CPL (#1), then commit and tag the ordered state in git.
5. #5, #6, #9, #10 and #11 don't affect the order. Do them when convenient.

## Verified OK

**Electrical:**
- **Power:**
  - +5 V from USB (D7) or J1 (F1 → D3) is OR-ed by Schottky diodes. The AP2112K-3.3 has its EN tied to VIN and 10 µF on input and output.
  - **LDO thermal:** θJA (SOT-25) is 184 °C/W per the Diodes datasheet. At an expected 0.15–0.2 A average that's 0.26–0.34 W, a rise of about 48–63 K, which is acceptable. It only approaches the 160 °C shutdown with a sustained load above ~0.4 A in a hot enclosure.
  - +12 V comes from J2 via F2 → D4. The relay coils (Axicom W11 V23101-D0006-A201, 12 V / 320 Ω, must-operate 8.4 V) get about 11.6 V after the Schottky. AO3400A drivers with 100 Ω gate resistor, 10 k pull-down and 1N4148W flyback diodes.
- **Strapping and pins:** IO0 BOOT (10 k + SW1), EN 10 k/100 n, IO3/IO45/IO46 unconnected, IO35/IO36 in use, so the module must be the quad-PSRAM N8R2 (assigned, C2913204).
- **USB:** CC 5k1 × 2, USBLC6-2SC6, both data pin halves tied, shield to GND.
- **Inputs:**
  - **NTCs:** FTH (R8 15 k 0.1 % → IO4) and tank (R9 10 k 0.1 % → IO5), each with 1 k / 100 n.
  - **3V3 sense:** R48/R49 0.1 % → IO2 (V_REF).
  - **Level switch:** 10 k pull-up, 1 k / 100 n → IO38.
  - **Flow:** 4k7 pull-up, 1 k / 1 n, 74LVC1G14 → IO7.
  - **Board B status lines:** E-Fast 1 k / 10 n → IO35, zero-cross 100 Ω / 100 p → IO36 (≈ 10 ns, OB-19), contactor aux → RXD0.
- **I²C:** single 4k7 pair (R6/R7 DNP). MCP9600 addresses 0x67 / 0x60.
- **J8 HMI:** +12 V on the edge pin next to GND, GND on 2/5/8, UART + GPIO through 1 k.
- **J9:** floating relay COM/NO pairs, 2 GND (pinout confirmed with board B).
- **SD:** 33 Ω × 4, pull-ups, 10 µ / 100 n. Card detect is a bodge wire to TP1/IO14 (decision).

**Model:**
- HN-1/HN-2/HN-4 SELV side complete.
- REQ-SYS-024, -066, -074 (relay NO, pending #2), -080 (FTH NTC insulation confirmed adequate), -081, -084, -086, -089 and -091 are all satisfied by the hardware as drawn.
- REQ-SUB-SEN-010: ungrounded probes (part spec).
- REQ-SYS-092/093: each MCP9600 sits next to its WAGO terminal on a slotted island (isothermal).

**Fabrication:**
- 4 layers, 1.6 mm. Default netclass 0.2 mm clearance / 0.2 mm track. Vias 0.6 / 0.3 mm (0.15 mm annular). Minimum PTH drill 0.2 mm (U5 thermal vias).
- All within JLC standard 4-layer capability.
- Antenna keep-out comes from the U5 footprint rule area and DRC is clean.
- 0 unconnected pads, schematic parity clean, zones refilled cleanly.

**BOM and ordering:**
- All 107 BOM symbols carry MPN, Manufacturer, Datasheet and Assembly fields.
- 74 placements across 17 JLC **basic** parts carry `LCSC`. 33 hand-placed parts carry `Hand LCSC`.
- JLCPCB Tools `project.db` matches the board 1:1 (111 rows, 0 mismatches), and the plugin option "add parts without LCSC" is off.
- `lcsc_order.csv`: J10 switched to USB4105-GF-A-120 (C5184243, in stock). WAGO 2060 must be bought outside LCSC.

**3D:** every placed part has a model, and the STEP export loads without errors.

## Settled by user decision (not findings)

These are excluded from scoring on purpose:
- **No** reset button, status LEDs or test points.
- **U5 decoupling** kept at C1 100 n + C2 10 µ. If brownouts appear, add 22 µF at pin 2.
- **SD card detect** is a bodge wire on this revision.
- **J8** carries 12 V + UART, no I²C.
- **Pressure sensor and HX711** are I²C, with no analog fallback.
- **Assembly:** JLCPCB basic parts only, everything else by hand.
- **Relays:** Axicom W11 (12 V) on the coil rail.
- **Closed earlier:** FTH II NTC insulation adequate (REQ-SYS-080); J9 pinout agreed with board B.
