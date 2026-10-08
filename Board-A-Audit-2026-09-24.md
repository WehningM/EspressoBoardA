# Board A audit — 2026-09-24

Scope: `Espressomaschine Board A.kicad_sch` (17.08.) and `.kicad_pcb` (28.08., 4-layer, 79 parts, KiCad 10)
checked against `BOARD-A.md` and `espresso.gaphor` (333 requirements, HN-1…HN-8, CON/MS blocks).
Method: netlist and geometry parsed from the files, KiCad ERC (39 violations), DRC with schematic parity
(49 violations, **0 parity issues**), and model requirements traced to the board. No design files were changed.

## Scoring

- **U** (urgency) 1–5: 5 = board dead / short / guaranteed M1 respin · 4 = required function missing (M2 respin)
  · 3 = spec/model deviation or degraded function · 2 = robustness, bring-up, fab · 1 = cosmetic
- **E** (effort) 1–5: 1 ≤ 15 min · 2 ≤ 1 h · 3 ≤ half day · 4 ≤ 1 day · 5 = needs an external decision
- **P = U / E**, ties broken by higher U. The ratio favours quick wins, so see the *critical path* note below.

## Ranked findings

| # | P | U | E | Finding & evidence | Ref |
|---|---|---|---|---|---|
| 1 | 5.0 | 5 | 1 | **+5V ↔ GND short**: an In2.Cu +5V track at x = 160.8 mm (y 74–79) runs through the GND via at (160.8, 79.0). A second GND via at (141.0, 84.5) is only 0.1 mm from a +5V In2 track. (DRC `shorting_items`, `clearance`) | DRC |
| 2 | 5.0 | 5 | 1 | **I²C pull-ups are 100 Ω, two pairs** (R4, R6 SCL→3V3; R5, R7 SDA→3V3), about 66 mA per line, so nothing can pull the bus low. Both MCP9600s and pressure are dead. Fix: R4/R5 = 4k7, reuse R6/R7 for #7 | Blk 3 |
| 3 | 5.0 | 5 | 1 | **USB-C only works in one plug orientation**: B6 (D+) and B7 (D−) unrouted, plus A4 (VBUS) and B1 (GND) not connected. This is the only programming port | Blk 13; DRC `unconnected_items` |
| 4 | 5.0 | 5 | 1 | **Flow signal never reaches the MCU**: net `FLW` = U4.4 + JP3.2 only | HN-2, REQ-SUB-SEN-004 |
| 5 | 3.0 | 3 | 1 | **No local 100 nF on MCP9600 U2/U3 or on 74LVC1G14 U4** (nearest cap > 35 mm away) | datasheets |
| 6 | 3.0 | 3 | 1 | **Label typos split nets**: `R1 OUT`(J9.8) ≠ `R1_OUT`(K1.1), `R2 OUT`(J9.7) ≠ `R2_OUT`(K2.1). The copper joins them, so DRC reports shorts and mask bridges | ERC/DRC |
| 7 | 2.5 | 5 | 2 | **HX711 (J7) sits on the I²C bus** (J7.2 = SDA, J7.3 = SCL). HX711 is not I²C: SCL idling high > 60 µs powers it down, and DOUT fights SDA. Needs 2 dedicated GPIOs + 100 Ω series | Blk 7, REQ-SUB-MAS-004, MS-4 |
| 8 | 2.0 | 4 | 2 | **GPIO budget is tight**: 14 used; the missing blocks need about 15 more (+1 for the FTH NTC, #32). Free: IO1, 2, 4–7 (ADC1), IO35–42, IO43/44 (UART0) = 16. IO35–37 are free only on a quad-PSRAM module, and the value doesn't fix the variant (N8R2). Any IO43/44 use must tolerate the ROM boot log | Blk 1, pin budget |
| 9 | 2.0 | 4 | 2 | **Card detect dangling**: `CARD_DET` only on IO14 (dangling 8.6 mm track). The symbol has no CD pins, and J13 pads 9/10 carry no net | Blk 10, REQ-SYS-086 |
| 10 | 2.0 | 2 | 1 | Starved thermal reliefs on GND pads: J10 A1/B12, U2 pads 1/5/18, U3 pads 1/5, C5 | DRC |
| 11 | 2.0 | 2 | 1 | J13 pad 10 touches the board edge (0 mm vs 0.5 mm rule) | DRC |
| 12 | 2.0 | 2 | 1 | **Spec errata**: the USB table says D+→GPIO19, but on the S3 GPIO19 = D− and GPIO20 = D+. The **schematic is correct**, so don't "fix" it to match the spec. Spec cites F-13/F-17/F-31, which don't exist in the model. Blk 3 still says "breakouts" | traceability |
| 13 | 2.0 | 2 | 1 | Power LED D2 on +12 V via 1 k (R19, 0805): 100 mW = 80 % of rating. On a 24 V coil rail, 0.48 W → burns | Blk 2 |
| 14 | 2.0 | 2 | 1 | ALERT_TEMP0: JP1.1↔U2.11 unrouted. MCP9600 ALERT pins are open-drain with no pull-ups | DRC |
| 15 | 2.0 | 2 | 1 | 2 status LEDs on GPIO missing | Blk 12 |
| 16 | 1.5 | 3 | 2 | **No reset button** (EN has only R28/C14) and no **J11** BOOT/EN header. Recovery only by BOOT + USB replug | Blk 1, Blk 13 |
| 17 | 1.5 | 3 | 2 | ESP32 decoupling is 1×100 n + 1×10 µ (C1/C2) vs spec 4×100 n + 100 µF bulk. Wi-Fi TX bursts → brownout risk | Blk 1 |
| 18 | 1.5 | 3 | 2 | **HMI J8: all 8 pins unconnected**, plus 8 dead-end traces (17–25 mm) ending near U5. J8 is a 1 mm FFC, not a header | Blk 11, HN-6, REQ-SYS-062 |
| 19 | 1.5 | 3 | 2 | **Polyfuses F1/F2 in the GND return** of J1/J2. GND is also shared via USB and J9, so the fuse is bypassed or ground lifts on trip. J1 is named "Supply 3V3" but feeds +5V | Blk 2 |
| 20 | 1.5 | 3 | 2 | BOM not orderable: D3/D4 "XX", D7 unspecified, fuses without I_hold, C3/C4 no voltage rating. Relay value "W11 V23101" doesn't match the HsinDa Y14 footprint (check pinout vs the bought part) | Blk 2/8 |
| 21 | 1.33 | 4 | 3 | **Block 9 absent**: ZERO_CR (J9.3), E_FAST_IN (J9.1), CNTCTR_AUX (J9.5) go nowhere: no 10 k pull-ups, no RC, no GPIO | Blk 9, REQ-SYS-081/-091, REQ-SUB-PROT-005, HN-1 |
| 22 | 1.33 | 4 | 3 | **Block 6 absent**: no tank NTC divider/100 nF and no level-switch pull-up/debounce. J5/J6 are used as TC terminals instead | Blk 6, REQ-SYS-066, REQ-SUB-THM-030, HN-2 |
| 23 | 1.33 | 4 | 3 | **Relay contacts not floating**: COM (pins 5/6) tied to +12 V, only NO (pin 1) goes to J9, 1 pin per relay. The spec wants isolated 2-pin pairs (J9: 11 pins, 1 GND); the board has 12 pins with 4 GND. The coil runs from +5 V, not the coil rail. The J9 pinout must be agreed with board B | Blk 8, J9, REQ-SYS-080 |
| 24 | 1.0 | 3 | 3 | No test points (9 signals + adjacent GND each) | Blk 12 |
| 25 | 1.0 | 3 | 3 | **Cold junction**: bare MCP9600s with WAGO terminals about 12 mm away (spec assumed breakouts). Any terminal↔die gradient becomes error (about 1 K per K). Make an isothermal island away from the LDO/relays | Blk 3, REQ-SYS-092/093, OB-21 |
| 26 | 1.0 | 2 | 2 | Pressure analog fallback missing: no DNP divider, no reserved ADC1 pin | Blk 4 |
| 27 | 1.0 | 1 | 1 | U5 thermal-via drills are below the 0.3 mm board rule (12×). Relax the rule to the fab's limit or change the footprint. U5 also differs from the library footprint | DRC |
| 28 | 1.0 | 1 | 1 | K2 not DNP / not labelled "spare actuator". J7 value is "I2C_HX771". Missing PWR_FLAGs (3 ERC errors). SBU pins lack no-connect flags | Blk 8, ERC |
| 29 | 0.8 | 4 | 5 | **Spec ↔ model conflict**: REQ-SYS-024 makes the FTH II integrated NTC (CON-2, 50 k) the controlled variable, but neither spec nor board has an input for it. REQ-SYS-084 says temperature accuracy must not depend on the MCU ADC, while the spec puts the tank NTC on ADC1. Decide: external ADC (e.g. ADS1115 on I²C) or change the requirement | model vs spec |
| 30 | 0.5 | 1 | 2 | USB pair is 0.3/0.2 mm over 0.1 mm prepreg, about 65–70 Ω differential rather than 90 Ω. Harmless at 12 Mbit/s | Blk 13 |

### Critical path

#8 (pin plan) and #29 (NTC/ADC decision) have low P but gate #21, #22, #18 and #26. Decide them first,
then do all the schematic wiring in one pass and reroute once. Close the schematic in KiCad before
editing (a `.lck` file is present).

### Suggested fix order

1. The 15-minute batch: #1, #2, #3, #4, #5, #6, #10, #11, #14.
2. Decisions: #8, #29, the J9 pinout with board B (#23), and the open BOM questions (#20).
3. Schematic pass: #7, #9, #21, #22, #16, #18, #15, #26, #24.
4. Layout pass: #25, #17, #19, then re-run ERC/DRC.

## Status re-check 2026-09-24 18:30 (sch 18:28, pcb 18:16)

- **Fixed:** #3 (B6/B7 and A4 now connected), #5 (C6, C7, C15 about 4 mm from U2/U3/U4), #6 (R1_OUT/R2_OUT;
  DRC shorts gone).
- **Fixed in the schematic only, PCB not yet updated:** #13 (R19 = 2k), J1 renamed "Supply 5V", J7 value, TP1.
- **Partly fixed:** #2: R4/R5 are now 4k7, but R6/R7 (100 Ω) still pull SCL/SDA to 3V3, so the bus is still dead.
  #3: the J10 A12/B1 GND pad has no connection. #9: CARD_DET now goes to TP1 only, still no detect switch.
  #24: TP1 is the first test point.
- **Unchanged:** #1 (the +5V/GND short is still at 160.8/79.0), #4, #7, #8, #10, #11, #14 onward.

## Status re-check 2026-09-24 19:18 (sch 19:11, pcb 19:17)

The HX711 is an **I²C module** (per the user), so #7 is withdrawn. Spec Blk 7 and model MS-3/MS-4 and
REQ-SUB-MAS-004 still describe a bare HX711 and need updating.

- **Fixed:** #1 short, #2 (no 100 Ω left), #3 (all USB pads), #4 (FLW → IO7), #10 thermal reliefs,
  #11 is now only J13 pad 10 at the edge, #13, #14 JP1 routed, #27 drill rule. Parity is clean, and DRC shows
  1 error + 4 cosmetic warnings.
- **Partly fixed:** #21: ZERO_CR→IO36, E_FAST_IN→IO35, CNTCTR_AUX→RXD0, but with no pull-ups, RC or
  protection. #18: J8 is wired, but with 4 GPIO, no SDA/SCL and **+12 V on J8.5**, between IO40 and IO41 at
  1 mm pitch.
- **New:**
  - N1: R6/R7 are now 4k7 in parallel with R4/R5, so the bus has 2.35 k plus the module and pressure
    pull-ups (over-strong bus).
  - N2: the remote I²C cable to the load cell shares the bus with the thermocouples.
  - N3: check the module's address (some default to 0x64–0x67, which clashes with U2 at 0x67), 10 SPS and the
    crystal (MS-3, F-31).

## Verified OK

USB CC 5k1 ×2, USBLC6-2SC6, VBUS Schottky, shield to GND; AP2112K wiring; EN 10 k / 100 n; BOOT 10 k
+ SW1; IO3/45/46 unconnected; I²C on IO8/9; SD 33 Ω ×4 + pull-ups + 10 µ/100 n; 5× AO3400A with 100 Ω
+ 10 k pull-down; flyback diodes. The relay NO contact is used, so the valve is unpowered at rest
(REQ-SYS-074, OI-1). MCP9600 addresses are distinct (ADDR = VDD → 0x67, GND → 0x60). The antenna
keepout is free of copper on all 4 layers. Reverse-polarity diodes and 220 µF per rail are present.
Schematic ↔ PCB parity is clean.

## Status re-check 2026-10-08 17:40 (sch 17:32, pcb 17:39)

ERC: 5 messages (4 errors, 1 warning). DRC: 23 violations, all silkscreen (1 text_thickness on 'V02',
plus silk overlaps around R8/R9/R40–R43/J11/J12/J14). **0 unconnected pads, 0 parity errors.**

- **Fixed:** #22 Block 6. J11 FTH NTC: R8 15k from 3V3 to the node, R41 1k, C20 100n → IO4 (`FTH_NTC`).
  J12 tank NTC: R9 10k, R40 1k, C19 100n → IO5 (`TANK_NTC`). J14 level: R42 10k pull-up,
  R43 1k, C21 100n → IO38 (`TNK_LVL`). #29 resolved by the REQ-SYS-084 change (2026-10-01).
  The J11/J12/J14 footprint parity mismatch is fixed. #23 relay contacts are floating COM/NO pairs.
  #21 Block 9 has pull-ups, series R and RC. N1: R6/R7 are DNP, so one 4k7 pair is left.
- **Decision, #9 card detect:** done with a **bodge wire** from the J13 card-detect pin to `CARD_DET`
  (TP1 / IO14) on this revision. Unrouted J13 pads 9/10 are intentional. Do not report this again.
  See BOARD-A.md Block 10.
- **Decision, #8 module variant:** **ESP32-S3-WROOM-1-N8R2**. Octal PSRAM is excluded because IO35/IO36 are
  in use. The -H4 is the fallback if board A runs hot. See BOARD-A.md Block 1. Still to do: put
  this in U5's Value/MPN field in the schematic.
- **New:** R8 (FTH divider) and R9 (tank divider) are plain "15k"/"10k". R9 is grouped in the BOM with
  the 1 % 10k pull-ups, so the 0.1 % / ≤25 ppm/K requirement from Block 6 would be lost at ordering.
  Give them a distinct value (e.g. "10k 0.1% 25ppm") and an MPN.
- **Still open:** #16 no reset button; #17 decoupling; #18/J8 has +12 V on pin 5 between GPIOs and no I²C;
  #19 polyfuses in the GND return; #20 BOM values (D3/D4 "XX", D7, F1/F2, C3/C4, relay
  "W11 V23101") and no LCSC/MPN; #24 test points; #25 cold-junction layout; #26 pressure analog
  fallback; #15 status LEDs; ERC PWR_FLAGs (U6 VIN, J13 VSS, #PWR06) and the TXD0 no-connect flag.
- **Model:** REQ-SYS-080 vs the FTH II NTC insulation (check OI-3). MS-3/MS-4 and REQ-SUB-MAS-004
  still describe a bare HX711.

### Addendum 2026-10-08 18:00 (sch 17:50)

- **Fixed:** #19. F1/F2 are now in the + line (J1.2→F1→D3→+5V, J2.1→F2→D4→+12V).
- **Decisions (user):** J8 carries +12 V and a UART (TX/RX), with **no I²C on purpose** (see BOARD-A.md Block 11).
  This replaces #18's SDA/SCL point. The open part of #18 is the pin order: today +12 V is on J8.5,
  between IO40 and IO41. The relays K1/K2 are confirmed correct as they stand, so #20's relay
  point is closed.
- **Model check, #15 status LEDs:** no model requirement asks for LEDs. REQ-SYS-062 puts all indication on the
  HMI display. LEDs are an optional bring-up aid only.
- **Decisions (user, 2026-10-08):** **no reset button** on EN. #16 is closed as "won't do": recovery is BOOT + power cycle.
  **No status LEDs** beyond D1/D2. #15 is closed as "won't do". Do not report either again.

## Status re-check 2026-10-08 19:00 (sch 18:20, pcb 18:35) and part assignment

Tests on the saved files before my edits:
- **ERC:** 4 errors + 1 warning. TXD0 has no no-connect flag; PWR_FLAG is missing on U6 VIN, J13 VSS and #PWR06; the GND/EXP name warning is harmless.
- **DRC:** 37 violations. 0 unconnected, and parity was clean. 31 silkscreen; 'V02' text too thin; 1 dangling 0.006 mm stub on the IO40
  track at (188.72, 49.78); and excluded items: J13 pad 10 at 0.435 mm from the edge, and the U5 library mismatch (thermal vias).
- **Netlist vs spec/model:** J8 matches the agreed pinout (12V/GND/TX/RX/GND/IO/IO/GND, 1k series). The IO2 3V3-sense divider
  (R48/R49 0.1 %, C22) is correct. F1/F2 are in the + line. FTH/tank/level are correct.

Changes made (schematic only; backup `Espressomaschine Board A.kicad_sch.bak-2026-10-08`):
- Added MPN / Manufacturer / LCSC / Datasheet and a detailed Description to all 109 placed symbols, chosen for **JLCPCB PCBA**
  (see BOARD-A.md "Part selection"). Values updated (e.g. D3/D4/D7 "XX" → SS34, C3/C4 → 220u 25V, fuses, LEDs green).
  JP1–JP3 and TP1 are excluded from the BOM. The netlist is unchanged by the field edits.
- U2/U3 → MCP9600-E/MX (the -I/MX is out of stock at JLC). The stale SnapEDA `MP` field is corrected.
- F1/F2 footprint 2010 → 1812 (no 2010 PTC at JLC). **PCB: re-place and reroute F1/F2.**
- J8 → TE 84952-8 (bottom contact). Its land pattern is identical to 84953-8. **Check the HMI FFC cable type.**
- **K1/K2: the parts the user has are Axicom W11 V23101-D0006-A201 = 12 V coil (320 Ω, must-operate 8.4 V)**, but the coils were fed
  from +5 V (they would never pull in), and the HsinDa Y14 footprint does not match the W11 (15.5×10.5 mm, pins on a 2.54 grid:
  12/11/7 over 1/2/6, 12.7×7.62 mm). Fixed: #PWR045/#PWR044 → +12V (only K1.3, K2.3, D5, D6 move; verified by netlist diff),
  and a new footprint `footprints:Relay_SPDT_Axicom_W11_V23101-D0xxx-A` (pads numbered as in the Y14 symbol: 1 NO = W11 pin 12,
  2 NC = pin 1, 3/4 coil = 11/2, 5/6 COM = 7/6). **PCB: K1 and K2 are 9.2 mm apart, but the W11 is 10.5 mm wide, so the relay area
  must be re-placed.** Verify NO/NC with a continuity meter before soldering (unpowered: COM 6/7 ↔ pin 1 should beep).
- **Decisions (user):** no test points (Block 12); U5 decoupling kept as is; pressure sensor and HX711 module confirmed I²C, with
  crystal, 10 SPS and no address clash, so no analog fallback (#26 closed).

## 2026-10-08 19:35: JLCPCB basic-only assembly

- **User confirmed:** FTH II NTC insulation is adequate (REQ-SYS-080 closed). J9 pinout is agreed with board B. Relay area re-placed and
  rerouted.
- **Tests:** ERC 3 errors + 1 warning (PWR_FLAG ×3, GND/EXP name). DRC 0 unconnected, parity clean, 76 cosmetic (74 silkscreen,
  'V02' text, 0.006 mm stub on the IO40 track at 188.72/49.78).
- **Attributes complete:** every BOM part has MPN, Manufacturer, Datasheet and Assembly. U2/U3 `MANUFACTURER` → `Manufacturer`.
  Datasheets added for F1, J5 and J6.
- **Assembly split:** 74 placements / 17 basic parts → `LCSC`; 33 hand parts → `Hand LCSC`. Wrote `jlcpcb/project.db` for JLCPCB
  Tools 2025.04.02, plus `jlcpcb/hand_assembly_parts.csv`. Plugin setting `lcsc_bom_cpl` set to false (backup
  settings.json.bak-2026-10-08). Backups: *.kicad_sch/.kicad_pcb.bak-2026-10-08-preJLC.
