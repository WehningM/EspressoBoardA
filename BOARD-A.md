# Board A — Control and Measurement

**SELV only. No mains, ever.** Nodes HN-1 (compute) + HN-2 (sensor frontend) + the SELV
side of HN-4 (isolation barrier). This is the whole of bring-up module M1, and it must be
fully testable on a desk with nothing but USB power.

The mains-side control header is designed in but left unconnected until M2.

---

## Block 1 — Compute

Bare **ESP32-S3-WROOM-1** module, reflow-soldered directly onto board A via its
castellated-pad footprint — not a DevKitC-class breakout. That means board A has to supply
everything a devkit would otherwise give for free: reset/boot circuitry and a programming
path. Block 13 stops being optional and becomes the sole way to flash and console into the
chip.

| Item | Qty | Spec | Note |
|---|---|---|---|
| ESP32-S3-WROOM-1 module | 1 | **ESP32-S3-WROOM-1-N8R2** (8 MB flash, 2 MB quad PSRAM), decided 2026-10-08, see below | Castellated pads, reflow-soldered |
| Decoupling, 100 nF | 4 | near the module supply pins | |
| Bulk cap, 100 µF | 1 | | |
| EN pull-up, 10 kΩ | 1 | EN to 3.3 V | No internal pull-up on the bare module — not optional |
| EN cap, 100 nF | 1 | EN to GND | Sets auto-reset timing for Block 13's programming circuit |
| ~~Reset button~~ | 0 | — | **Not fitted (user decision 2026-10-08).** Recovery = hold BOOT (SW1) while power-cycling; with J1 connected, J1 must be disconnected too |
| BOOT (GPIO0) pull-up, 10 kΩ | 1 | GPIO0 to 3.3 V | Must read high at reset for normal boot |
| BOOT button | 1 | momentary, GPIO0 to GND | Held low at reset to force download mode |

**Module variant: ESP32-S3-WROOM-1-N8R2 (decided 2026-10-08).** Reasons:

- **Quad, not octal PSRAM, is a hard constraint.** Octal PSRAM (N8R8, N16R8, N16R16VA)
  consumes GPIO33–37. The schematic uses IO35 (E_FAST_IN) and IO36 (ZERO_CR), and IO37 is
  kept as a spare. On an octal module, the zero-cross clock (REQ-SYS-081, -091, AR-1) and the
  E-Fast trip status (REQ-SUB-PROT-005) would not work.
- **Temperature rating.** The N4/N8/N16 and N4R2/N8R2 variants are rated −40…85 °C. The
  octal variants are rated −40…65 °C only, which is marginal inside a machine with a 2 kW
  heater.
- **8 MB flash.** This is enough for the application, the web UI for profile editing
  (REQ-SYS-077) and the NVS calibration partition (REQ-SYS-064, -089). Profiles live on the
  SD card (HN-7). Firmware updates go over the wired port (REQ-SYS-065), so no dual OTA
  partition is needed, but it still fits if one is wanted later.
- **2 MB PSRAM.** This is headroom for the HMI framebuffer (OI-8 still open) and the web
  server's buffers. It costs no GPIO: the quad PSRAM shares the flash bus and is not broken
  out.
- **Fallback:** if the measured air temperature at board A's location is above about 75 °C,
  use **ESP32-S3-WROOM-1-H4** instead (−40…105 °C, 4 MB flash, no PSRAM). It has the same
  footprint and pinout.
- Keep the PCB-antenna WROOM-1, not the -1U. That only changes if the enclosure turns out to
  be metal without an antenna cutout.

Unavailable regardless: GPIO0/3/45/46 (strapping), GPIO19/20 (USB), and the SPI flash
block around GPIO26–32.

**GPIO45/GPIO46 are also strapping pins** (VDD_SPI voltage select and boot-mode/ROM-message
select). Leave them unconnected, per Espressif's reference design — don't let the I2C bus,
a pull-up, or anything else land on them.

**Antenna keepout.** The WROOM-1 carries an onboard PCB antenna. Keep copper — pour,
traces, components — out of the keepout area on that edge per the module datasheet, and
don't route a metal enclosure wall over it without a cutout.

## Block 2 — Power

| Item | Qty | Spec | Note |
|---|---|---|---|
| 5 V input connector | 1 | screw terminal or barrel | Not needed during M1 — USB is enough |
| Coil rail input | 1 | **12 V (decided 2026-10-08)** | Feeds the K1/K2 coils (Axicom W11, 12 V) and the HMI on J8 |
| Reverse-polarity MOSFET or Schottky | 2 | one per rail | |
| Polyfuse | 2 | one per rail | |
| Bulk caps, 220 µF | 2 | one per rail | |
| Power LED + resistor | 1 | | |
| 3.3 V LDO, AP2112K-3.3 | 1 | SOT-23-5, 600 mA | Board-A generates its own 3.3 V — see below |
| LDO input/output caps, 1 µF ceramic | 2 | | Per AP2112K-3.3 datasheet |

**3.3 V is generated on board A.** A bare WROOM-1 module has no onboard regulator at all —
it needs 3.3 V fed to it directly, so this was never optional and there's no devkit-regulator
ambiguity to route around. Board A's peripherals (2× MCP9600, SD card, HX711, pressure
sensor, flow pull-up) share that same rail, fed from 5 V (USB during M1, the input connector
from M2).

An LDO, not a switching DC-DC, is the right topology here: the drop is only 5 V → 3.3 V,
the load is small (well under the 500 mA budget below), and I2C plus the analog thermocouple
and NTC front ends benefit from the lower noise floor. **AP2112K-3.3** (SOT-23-5, 600 mA,
low-noise) is the pick — cheap, common on JLCPCB assembly, and gives headroom over budget.
AMS1117-3.3 (SOT-223) is an easier-to-hand-solder alternative if you're not getting the
board assembled — bigger package, a bit more dropout and noise, both irrelevant at this
current.

Budget check: two MCP9600s, the SD card and the HX711 together are comfortably within a
typical 500 mA LDO, but confirm against your actual measured draw.

**Size the coil rail for two relays**, since the spare may be populated later.

## Block 3 — Temperature (TC1 heater outlet, TC2 brew head)

| Item | Qty | Spec | Note |
|---|---|---|---|
| MCP9600 breakout | 2 | on female headers | Breakouts, not bare chips — cold junction and TC connector already solved |
| Female headers | 2 sets | | |
| J-type thermocouple | 3 | Class 1, **ungrounded junction**, MI stainless sheath. 1.5 mm for TC1, 3 mm for TC2 | Third is for measuring OB-29 during M5 |

**The two must sit at different I2C addresses** — set by solder jumper on the breakout.

**I2C pull-up trap:** each breakout carries its own pull-ups. Two breakouts plus the
pressure sensor in parallel gives an over-strong bus that will fail intermittently and
look like a wiring fault. Cut the jumpers on all but one device, or remove them all and
fit a single pair (4.7 kΩ) on board A. The single-pair-on-board-A option is cleaner.

**No dedicated board-A connector for the thermocouples.** The MCP9600 breakout carries its
own TC screw terminal — that's what "cold junction and TC connector already solved" means —
and the breakout plugs into board A's female headers directly, not through a panel
connector. So the thermocouple's own lead is what runs from TC1/TC2's physical location
(heater outlet, brew head) all the way back to the breakout's terminal on board A.

**That lead must stay genuine J-type thermocouple wire (or matched compensating cable) for
its entire length.** Any generic connector spliced into the run introduces a second,
uncompensated junction at a different temperature than the MCP9600's own cold junction,
corrupting the reading. If you want a disconnect point at the enclosure wall for
serviceability, it has to be a J-type-matched thermocouple connector (mini or standard,
colour-coded per IEC 584 or ANSI) — not a generic pin header, and not currently in this BOM.

## Block 4 — Pressure

| Item | Qty | Spec |
|---|---|---|
| 4-pin connector | 1 | 3.3 V, GND, SDA, SCL |
| Divider footprint | 1 set | unpopulated, see below |

HN-2 specifies pressure over I2C, and that's the plan. But many cheap espresso-range
transducers are **analog** ratiometric 0.5–4.5 V at 5 V supply, which would need an ADC1
channel and a divider instead. Fit the connector plus an unpopulated resistor-divider
footprint and reserve an ADC1 pin, so either sensor works without a respin. Confirm which
type you're buying before layout.

## Block 5 — Flow (Digmesa)

| Item | Qty | Spec | Note |
|---|---|---|---|
| 3-pin connector | 1 | V+, GND, signal | |
| Pull-up, 4.7 kΩ | 1 | **to 3.3 V** | If the sensor is open-collector, pulling up to 3.3 V does the level shift for free |
| RC filter, 1 kΩ + 1 nF | 1 | | Keep it small — see below |
| Schmitt buffer footprint | 1 | 74LVC1G14, optional | Fit if edges look soft on the scope |

REQ-SUB-SEN requires ticks counted in an interrupt with none lost, so edge quality matters.
Filter enough to reject switching noise and no more.

## Block 6 — Tank and FTH NTC

| Item | Qty | Spec |
|---|---|---|
| 2-pin connector | 3 | tank NTC, FTH II NTC, level switch |
| NTC divider resistor | 2 | 0.1 %, ≤25 ppm/K, from +3V3 to the ADC node; the NTC goes from the node to GND. FTH (50k, CON-2): 15k. Tank: 10k for a 10k NTC |
| Series resistor, 1 kΩ | 2 | between the connector and the ADC pin |
| Filter cap, 100 nF C0G/X7R | 2 | at the ADC pin |
| Pull-up + debounce RC | 1 set | level switch, 10k / 1k / 100 nF |

Both NTCs go to **ADC1** channels (FTH → IO4, tank → IO5). ADC2 is unusable while Wi-Fi is
active (REQ-SYS-084). REQ-SYS-084 was relaxed on 2026-10-01 to allow the MCU ADC1 for the
NTCs (option B of audit item #29), with factory calibration (`adc_cali` curve fitting) and a
per-board offset calibration against a reference thermometer. The thermocouples stay on the
MCP9600s.

Keep the divider inside the ADC's best range: use 12 dB attenuation but stay between about
0.15 V and 2.5 V. The FTH divider gives 2.5 V at 25 °C, 0.73 V at 93 °C (about 17 mV/K) and
0.39 V at 120 °C. An open sensor reads 3.3 V and a shorted one 0 V, so firmware can flag both
as faults and must then switch the heater off. The divider is fed from +3V3, so 3V3 tolerance and ripple land directly in
the reading. Firmware should average many samples (e.g. 64), and the 3V3 rail must stay quiet.

## Block 7 — Mass (interface to the remote HX711)

| Item | Qty | Spec |
|---|---|---|
| 4-pin connector | 1 | 3.3 V, GND, SCK, DOUT |
| Series resistors, 100 Ω | 2 | on SCK and DOUT |

The HX711 module itself lives **at the load cell**, not on this board — analog load-cell
wiring wants to be short, and the digital side doesn't care about cable length. Buy it as
a breakout.

Two things to verify on whatever module you buy:
- **Crystal clock, not RC.** F-31 records that the filter null only stays on 50 Hz with a
  crystal, and flags the frequency as recalled rather than read from the datasheet. Worth
  reading properly now.
- **RATE pin accessible** so you can select 10 SPS (F-17).

## Block 8 — Actuator drivers (SELV side of HN-4)

Five channels: heater SSR, pump SCR gate, valve relay, spare actuator, contactor hold. All
five are low-voltage signals that leave the board through J9 to board B, which does the
actual mains-side switching. **None of these MOSFETs or relay contacts ever carry mains
current on board A** — that would break the "SELV only, no mains, ever" rule this whole
board exists to satisfy.

| Item | Qty | Spec | Note |
|---|---|---|---|
| Logic-level N-MOSFET | 5 | AO3400 or IRLML2502, SOT-23 | **Logic-level** is the critical word |
| Gate pull-down, 10 kΩ | 5 | **all populated** | Defines the pin during reset, boot and flashing |
| Gate series resistor, 100 Ω | 5 | | |
| Flyback diode | 2 | 1N4148 or Schottky | Relay channels |
| Relay | 1 of 2 | **Axicom/TE W11 V23101-D0006-A201** (decided 2026-10-08): SPDT, 12 V coil 320 Ω / 450 mW, must-operate 8.4 V, AgPd+Au 1.25 A. Coil on **+12 V**, not +5 V. Footprint `footprints:Relay_SPDT_Axicom_W11_V23101-D0xxx-A` | Contacts carry J9's floating valve/spare signal onward to board B (see below) — the mains rating is for reliability, not because mains current runs through it here. **Second footprint DNP** — label it "spare actuator", not "valve 2" |

The pull-downs are the safety-relevant part. Firmware is exactly what is absent during a
reset, so "off by default" has to be true in copper.

**Mains switching for the valve and spare actuator happens on board B, not here.** The
relay on board A only adds a robust switch stage to the low-voltage signal it forwards
through J9 — board A's own conductors never see mains voltage or current, on any of the
five channels.

**Valve and spare need a genuinely floating contact on the board-B side, unlike the other
three channels.** Their two J9 pins each form an isolated pair (relay common + relay NO/NC)
carrying no reference to board A's ground — they must **not** be wired to J9's shared gnd
pin the way SSR, SCR gate and contactor hold are. This is also why valve and spare keep
their relay footprint instead of collapsing into a sixth plain MOSFET stage: a MOSFET's
drain-source path is inherently ground-referenced to board A, so it can't produce a floating
contact without an added optocoupler or isolated driver stage. One physical connector (J9)
still serves all five channels — it just carries a mix of ground-referenced and floating
pin pairs, not eight identical signal pins.

## Block 9 — Isolated input receivers

Three logic-level inputs arriving from board B: zero-cross, E-Fast trip, contactor aux
state. **The optocouplers live on board B** — board A sees SELV on both sides of the
connector, which is how REQ-SYS-080 stays true by construction.

| Item | Qty | Spec |
|---|---|---|
| Pull-up, 10 kΩ | 3 | |
| RC filter | 3 sets | |

**Keep the zero-cross RC minimal.** Every nanosecond of filter delay on that edge lands
directly in the OB-19 jitter budget. If it needs cleaning up, a Schmitt buffer is better
than a bigger capacitor.

## Block 10 — Storage (HN-7)

| Item | Qty | Spec | Note |
|---|---|---|---|
| microSD socket | 1 | SPI mode, **with card-detect** | Card-detect serves REQ-SYS-086 |
| Series resistors, 33 Ω | 4 | CLK, CMD, DAT, CS | |
| Pull-ups, 10 kΩ | 2 | CS, MISO | |
| Decoupling, 10 µF + 100 nF | 1 set | local to the socket | SD cards draw current in bursts |

Not skippable on a bare module — there's no devkit SD socket to inherit.

**Card detect is a bodge wire on this revision (decided 2026-10-08).** The card-detect switch
pads of J13 (Hirose DM3D-SF, pads 9/10) are not routed. After assembly, a wire connects the
socket's card-detect pin to the `CARD_DET` net (IO14, exposed on TP1). This is intentional,
and missing CD copper on J13 must not be reported as a defect for this revision. Firmware
still uses IO14 for REQ-SYS-086 / REQ-SUB-STO-006: enable the internal pull-up, and treat
the switch state as "card present / absent". Route it properly on the next board revision.

## Block 11 — HMI connector (HN-6)

8-pin header: 3.3 V, GND, SDA, SCL, and 4 GPIO. Deliberately generic — OI-8 hasn't settled
the display type, and this keeps every option open without committing the board.

**Revised 2026-10-08 (user decision):** J8 is a 1 mm FFC (TE 84953-8). It carries **+12 V**
power and a **UART (TX/RX)** link to the HMI. There is **no I²C** on J8.

- **Why 12 V:** for the same power, 12 V needs less than half the current of 5 V, which
  suits the thin FFC conductors if the HMI ever needs more power.
- **UART:** use UART1 through the GPIO matrix on the J8 GPIOs (IO39–42). Do not use UART0
  (IO43/44): it carries the ROM boot log, and RXD0 is already CNTCTR_AUX.
- **Pinout rules:** +12 V sits on an edge pin, with only GND next to it. Every signal line
  gets a 1 kΩ series resistor at board A. The HMI regulates its own logic supply from 12 V,
  so 3V3 is not exported on J8. This keeps HMI load and noise off the NTC reference rail
  (Block 6). The HMI must use 3.3 V logic.
- **12 V is only present when J2 is powered,** so the HMI is dark during USB-only M1 bring-up.

## Block 12 — Test points

You will be scoping this board through all of M1 and M2, so make it easy:

**No dedicated test points on this revision (user decision 2026-10-08)**; probe at resistor and connector pads. Original list for reference:

Signals: zero-cross in · SCR gate out · SSR out · SDA · SCL · HX711 SCK · HX711 DOUT ·
3.3 V · 5 V

**Put a ground pad beside each one.** A single distant ground clip adds ringing that will
have you chasing a signal-integrity problem that only exists in your probe.

~~Plus 2 status LEDs on spare GPIO.~~ **No status LEDs (user decision 2026-10-08).** The model requires none: REQ-SYS-062 puts all indication on the HMI. Only the D1 (5 V) and D2 (12 V) power LEDs are fitted.

## Block 13 — Programming and debug USB (mandatory)

A bare WROOM-1 module has no onboard USB-serial bridge and no USB connector of its own.
This block is board A's **only** way in — not a fallback for an unreachable devkit port,
since there is no devkit. Firmware flashing, console and JTAG all ride this one connector.

| Item | Qty | Spec | Note |
|---|---|---|---|
| USB-C receptacle | 1 | 16-pin USB 2.0, horizontal top-mount, SMT pins + 4 THT shield legs | GCT USB4085-GF-A. XKB U262-161N-4BVC11 is the LCSC/JLCPCB equivalent |
| CC pulldown, 5.1 kΩ 1% | 2 | **one per CC pin** | |
| Schottky or solder jumper | 1 | VBUS to the 5 V rail | |
| ESD array | 1 | USBLC6-2SC6 across D+/D−, optional | The connector that gets handled most |

**Skip the 6-pin USB-C variants.** The common ones are power-only, with no D+/D−.

| Pin | Connect to |
|---|---|
| VBUS (A4/B4, A9/B9) | 5 V rail via the Schottky or jumper |
| GND (A1/B1, A12/B12), shell | GND directly |
| CC1 (A5) | 5.1 kΩ to GND |
| CC2 (B5) | 5.1 kΩ to GND |
| D+ (A6 **and** B6) | GPIO19 |
| D− (A7 **and** B7) | GPIO20 |
| SBU1/SBU2 (A8/B8) | no connect |

**Two separate CC pulldowns.** Sharing one resistor between CC1 and CC2 makes the source
misread the orientation, and the port either fails to enumerate or fails to power.

**Both halves of each data pin tied together** (A6+B6, A7+B7). That is what makes the cable
reversible.

**VBUS backfeed.** Block 2 already has a reverse-polarity MOSFET on the 5 V input. The
Schottky or jumper here stops USB power and that rail fighting each other. During M1, USB
alone is enough anyway.

Route D+/D− as a short 90 Ω differential pair straight to the header pins. The ESP32-S3
native USB needs no series resistors. Shield goes to GND directly — the 1 MΩ ∥ 4.7 nF
arrangement is for boards with a separated chassis ground, which this one is not.

**Thermal relief on the four THT shield-leg holes.** Tied straight into the ground pour they
sink heat faster than a small iron supplies it, and that is where the connector's mechanical
strength lives.

**Why the recovery path still works even though this is the only port.** This connector is
the chip's built-in USB-Serial-JTAG — download, console and JTAG debug on one cable. Firmware
that misconfigures GPIO19/20 or disables the peripheral takes down the *application's* use of
the port, but holding BOOT (GPIO0, Block 1) low at reset forces the ROM bootloader, which
resets the USB peripheral to its default state regardless of what the last firmware did.
That's exactly why Block 1's BOOT button isn't optional on a bare module: it's the only way
back in if firmware ever breaks its own USB path, so it needs to stay reachable once the board
is installed. No reset button is fitted (user decision 2026-10-08), so the "reset" is a power
cycle while BOOT is held: unplug USB **and** J1.

---

## Connector summary

| # | Purpose | Pins |
|---|---|---|
| J1 | 5 V in | 2 |
| J2 | Coil rail in | 2 |
| J3 | Pressure sensor | 4 |
| J4 | Flow meter | 3 |
| J5 | Tank NTC | 2 |
| J6 | Tank level | 2 |
| J7 | HX711 (remote) | 4 |
| J8 | HMI | 8 |
| **J9** | **Control header to board B** | **11** |
| J10 | USB-C, programming and debug (Block 13) | 16 |
| J11 | FTH II NTC (Block 6) → IO4. Replaces the earlier BOOT/EN header plan | 2 |
| J12 | Tank NTC (Block 6) → IO5 | 2 |
| J13 | microSD (Block 10). Card detect via bodge wire to TP1/IO14 | — |
| J14 | Tank level switch (Block 6) → IO38 | 2 |

As built (2026-10-08), J5/J6 are the thermocouple terminals TEMP0/TEMP1 rather than the tank
NTC and level switch listed above.

J9 carries 11 pins: 3 ground-referenced outputs (SSR, SCR gate, contactor hold), 3
ground-referenced inputs (zero-cross, E-Fast trip, contactor aux), 1 shared gnd pin for
those six, and 2 floating relay-contact pairs (valve, spare) carrying no reference to board
A's ground at all. Board B does the actual mains switching on the far side of this header;
nothing on J9, or anywhere on board A, carries mains current.

## Pin budget

About 25 GPIO in total — outputs 5, isolated inputs 3, I2C 2, flow 1, NTC 1, tank level 1,
HX711 2, SD 4–5, HMI 4, LEDs 2. The bare WROOM-1 breaks out comfortably more than that
(45 GPIO on the chip, minus the reserved ranges), with room left for the strapping and
flash pins you must avoid.

I2C: **GPIO8/GPIO9 (SDA/SCL)** is the recommended pair — free on every WROOM-1 flash/PSRAM
variant, outside all reserved ranges, and matches the ESP-IDF/Arduino default so example
code and breakout libraries need no pin reconfiguration.

Block 13 costs nothing against this budget: GPIO19/20 are already listed as unavailable in
Block 1, and BOOT/EN are not GPIO you allocate.

The actual pin assignment is the next step, once the module's flash/PSRAM variant is fixed.

---

## Part selection (2026-10-08)

Board A is assembled by **JLCPCB PCBA with basic parts only** (decided 2026-10-08). Everything else is hand-assembled. The split is encoded in the schematic/PCB fields: basic parts carry `LCSC` (read by the JLCPCB Tools plugin); hand-assembled parts carry `Hand LCSC` instead (ignored by the plugin); every part has `Assembly` = "JLCPCB basic" or "Hand (extended/THT/user-supplied)". The plugin DB is `jlcpcb/project.db` (17 unique basic parts, 74 placements). The plugin option "add parts without LCSC to BOM/CPL" is OFF. The shopping list for the hand-assembled parts is `jlcpcb/hand_assembly_parts.csv`. No basic drop-in exists for U1, U4, U6, F1/F2, R8/R9/R48/R49 (0.1 %), SW1 or the connectors without footprint changes.

Originally: Every placed part carries `MPN`, `Manufacturer`, `LCSC` and `Datasheet` fields in the
schematic, and the Description field also repeats the datasheet link. Notable choices:

| Ref | Part | Why |
|---|---|---|
| U5 | ESP32-S3-WROOM-1-N8R2 (C2913204) | Block 1 |
| U2/U3 | **MCP9600-E/MX** (C220851) | The -I/MX has 0 stock at JLC. The -E/MX is the same die and MQFN-20 package, rated -40…125 °C |
| R8 / R9, R48, R49 | Yageo RT0805BRD0715KL / RT0805BRD0710KL, 0.1 % 25 ppm | NTC dividers and 3V3 sense (Block 6, REQ-SYS-084). Never substitute 1 % parts |
| F1 / F2 | SMD1812P110TF/16 (1.1 A) / SMD1812P075TF/24 (0.75 A) | Footprint changed 2010 → **1812**, because JLC stocks no 2010 PTCs. F2 ≤ the 1 A FFC contact, and HMI load is ≤ 0.5 A |
| J8 | **TE 84952-8** (bottom contact, C590895) | 84953-8 is out of stock. Identical land pattern. The FFC cable must be chosen so that board pin 1 (+12 V) lands on HMI pin 1 |
| J13 | Hirose DM3D-SF (C719027) | Fits the footprint and has an N.O. card-detect switch. Card detect is a bodge wire on this revision (Block 10) |
| K1/K2 | Axicom W11 V23101-D0006-A201, **user-supplied, hand-solder** (no LCSC) | 12 V coil on +12 V. New footprint, see Block 8 |
| D3/D4/D7 | SS34 (C8678) | 40 V 3 A Schottky |
| C3/C4 | Rubycon 25YXF220MEFC8X11.5 | 220 µF 25 V, D8/P3.5, fits the D10/P3.5 footprint |
| D1/D2 | KT-0805G green | Power LEDs only (no status LEDs, user decision) |

U5 decoupling stays as drawn (C1 100 nF + C2 10 µF; user decision 2026-10-08). If brownout resets appear at bring-up, add 22 µF at U5 pin 2.

---

## Open decisions that change this BOM

| Question | Affects |
|---|---|
| ~~Valve relay coil voltage?~~ **Decided 2026-10-08: 12 V** (Axicom W11 V23101-D0006-A201) | Block 2 rail voltage and sizing |
| ~~Pressure sensor: I2C or analog?~~ **Confirmed I²C (2026-10-08)**, no analog fallback fitted | Block 4, and whether an ADC1 pin is consumed |
| Digmesa output type and supply voltage? | Block 5 pull-up rail and level shifting |
| ~~HX711 module: crystal or RC clock?~~ **Confirmed (2026-10-08):** I²C module, crystal, 10 SPS, address clear of 0x60/0x67 | F-31 validity — the whole 50 Hz rejection argument |
| ~~Module variant (quad or octal PSRAM)?~~ **Decided 2026-10-08: N8R2** (Block 1) | Available GPIO, then the pinout |
| Where does Block 13's USB-C receptacle land relative to the enclosure wall? | Front-panel cutout, and whether BOOT/EN (J11) also need to reach outside |

## One honest note on sequencing

Four of the five questions above are interface details that a PCB has to commit to. If
you'd rather not risk a respin, the low-cost path is to breadboard the uncertain blocks
(pressure, flow, HX711) against a cheap devkit or WROOM-1 breakout first — separate from
board A's bare-module layout — confirm the interfaces, and lay out board A once. The
measurement blocks are the ones worth being sure about, since proving measurement is what
M1 exists for.
