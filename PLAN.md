# lorawan-airrh — Program Plan

Last revised 2026-09-23. Derived from `Abstract.md`, `Design Requirements.md`, the
datasheets in `Datasheets/`, and a direct audit of the KiCad project (ERC report, exported
netlist, 3D renders). Supersedes the working history kept at `build/PLAN-working-history.md`.

> ## STANDING CONSTRAINT — NO ROUTING
> Set by the engineer 2026-09-23. Agent work stops at **optimized component placement**.
> Do not lay any track segments. Deliver placement, zones/keepouts, mounting holes,
> fiducials and a synced netlist; **final routing is done by the engineer.**

## 0. Product definition

The gas sensors cap the operating envelope at +10 to +40 C (STCC4) — a 30-degree indoor
window, versus -40 to +85 C for the base node. That constraint, together with the 22x power
gap on VOC/NOx, splits this into two products rather than one:

| | **SKU A — Environmental** | **SKU B — Air Quality** |
|---|---|---|
| Channels | Temp / RH | Temp / RH, CO (confirmed), VOC in **ppb** (confirmed), CO2 / NOx TBD |
| Power | 2x AAA alkaline, 5+ years | Mains / external DC |
| Temp range | **0 to +50 C** (see 0.1) | +10 to +40 C |
| Target | Cold storage, greenhouses, outdoor, remote | Indoor warehouse, FMCG handling, IAQ |
| Board | ~36 x 71 mm, slim | ~46 x 88 mm, vented gas chamber |
| Cost | sub-$50 target holds | ~$40 higher in sensors |
| Status | **This project — in progress** | Derivative, not started |

### 0.1 Temperature envelope — CORRECTED 2026-09-23

An earlier version of this plan claimed SKU A was **-40 to +85 C**. That was wrong: the
**GDEY0213B74 panel is TOPR 0 to +50 C, TSTG -25 to +70 C** (datasheet operating-conditions
table). The e-paper panel reintroduces exactly the limitation the SKU split was meant to
avoid, so the split can no longer be justified on temperature for the display.

Engineer relaxed the requirement the same day: **-40 C is not needed; 0 C, or down to
-10 C, is sufficient.** That makes the panel acceptable.

Per-block limits as actually specified:

| Block | Operating | Note |
|---|---|---|
| SHT45 | -40..+125 C | |
| nPM2100 | -40..+105 C | |
| RAK811, W25Q16JV, LIS2DW12 | -40..+85 C | |
| **GDEY0213B74 panel** | **0..+50 C** (storage -25..+70) | binding constraint |

**Design consequence:** logging, flash and LoRa all work to -40 C; only the *display refresh*
is limited. Firmware should gate refreshes below 0 C using the onboard SHT45 reading - e-ink
is bistable, so the last image stays visible with no power. Down to -10 C the panel is well
inside its -25 C storage limit, so this costs nothing but a firmware interlock.

**The SKU split still stands, but on power, not temperature:** SGP41 VOC/NOx is ~22x the
2x AAA annual budget regardless of temperature. That was always the stronger argument.

Common to both: RAK811 LoRaWAN OTAA, GDEY0213B74 2.13" e-ink, W25Q16JV flash logging,
SHT45, TagConnect SWD, 4-layer, U.FL antenna.

**SKU A adds no new sensors.** It is the board that already exists, plus the fixes below.

## 1. SKU A — current state

**Schematic** — single sheet, 40 placed parts, 50 ERC violations.

| Block | Part | State |
|---|---|---|
| PMIC | nPM2100-QEAA (U3) | Power path correct: L1 2.2uH, C11 10u / C15 22u / C13 2.2u / C16 2.2u all match datasheet externals. Control side not wired. |
| LoRa | RAK811-HF-EU868 (U2) | Powered, SWD, BOOT0, RST, I2C1 to sensor. No RF path. 15 GPIOs unassigned. |
| Sensor | SHT45-AD1B (U1) | Wired on I2C1 + C18 decoupling. Done except pull-ups. |
| Flash | W25Q16JVSS (U4) | Only VCC/GND. All 6 SPI pins floating. |
| Display | J2 24-pin FPC + external booster | Booster chain built; control loop open, no MCU connection. |
| Debug | J3 TagConnect TC2030 | Done. |

**PCB** — 36.0 x 71.0 mm, 2-layer, 1.6 mm. 23 of 40 footprints placed, **0 nets defined,
0 track segments**, 3 unfilled zones, 2 stray vias, no mounting holes, no fiducials. The
netlist has never been pushed to the board. Effective state: rough placement only.

## 2. Blocking defects

| # | Defect |
|---|---|
| B1 | **No antenna path.** `RF_OUT` (U2 pin 33) is a one-node net. No match, no connector, no feed. |
| B2 | **Flash has no bus** — 6 SPI pins float. RAK811 exposes *no usable hardware SPI*: SPI1 (PA4-7) is internal to the SX1276, SPI2's clock PB13 is not on the pinout. Must be bit-banged. |
| B3 | Display CS/DC/RST/BUSY/SCK/SDA all dangle at the connector, never reach the MCU. |
| B4 | **Zero I2C pull-ups on the board.** nPM2100 datasheet requires them explicitly. Bus cannot work. |
| B5 | `nPM_SDA`/`nPM_SCL` unconnected — the PMIC is unreachable. |
| B6 | SHPHLD (U3.4) floating. Needs a button to GND; abs-max 1.9 V so it cannot tie to the rail. |
| B7 | Display `RESE` (booster current sense) floating — charge-pump control loop is open. |
| B8 | **Rail conflict.** VSET NC boots the boost at 3.0 V; RAK811 spec is 3.15-3.45 V. VSET selects only 3.0 or 1.8 V — 3.3 V needs an I2C write from the RAK811 itself. Firmware must raise the rail before first TX. |
| B9 | 13 wrong/missing footprints: C1-C7, C19-C21 assigned `D_SOD-123`; R1/R2 assigned SOT-323; L2 blank. Same 13 parts absent from the PCB. |
| B10 | PCB out of sync — zero nets, zero tracks. |
| B11 | Panel is 29.2 x 59.2 x 1.0 mm and needs a flat face. Resolved by D4 below. |
| B12 | Battery pulse capability. Resolved by D1 below. |

### Non-blocking cleanup
- U3 pins 14/15 (VINT) trip a power-out-to-power-out ERC error — net tie or documented exclusion.
- `SUDOCORP:nPM2100-QEAA` symbol has drifted from the global library copy.
- 3 off-grid wire/pin endpoints.
- `DISP_BS1` collides with GND, `DISP_VCI` with +3V0 (merges correct, labels redundant).
- `DISP_VPP`, J2 pin 1, J2 pin 4 need explicit no-connect flags (datasheet: "keep open").
- No MPN / manufacturer / LCSC fields — BOM is not orderable.
- No mounting holes, fiducials, or edge chamfers.
- No project-local libraries; depends on the global `SUDOCORP` lib.
- KiCad holds lock files on `.kicad_sch` / `.kicad_pro` — close it before edits.

## 3. Resolved decisions

| # | Decision | Choice |
|---|---|---|
| D1 | Battery | **2x AAA alkaline** |
| D2 | Antenna | **U.FL + external pigtail** |
| D3 | Stackup | **4-layer** |
| D4 | Mechanical | **Panel on front, over an enclosure cutout** |
| D5 | Gas sensing | **Deferred to SKU B** (separate board) |

**D1.** ~1200 mAh, 3.0 V fresh down to \~1.8 V — inside the nPM2100's 0.7-3.4 V boost window
across the whole discharge curve. Alkaline AAA sources 100+ mA without sag, closing B12: no
bulk reservoir, and 20 dBm TX stays available. Holder is ~44.5 x 21 x 11 mm and claims most
of the back face. BT1 needs a new 2-cell symbol and footprint; the CR2450 holder is dropped.

**D2 + D3.** With 4 layers (0.2 mm L1-L2 prepreg typical at JLC) a 50 ohm microstrip over
solid L2 ground is \~0.35 mm wide, routing comfortably in 36 mm — versus ~2.9 mm on the old
2-layer stack. Plan: pi-match footprint (0R + DNP initially) at U2 pin 33, short
ground-flanked 50 ohm run to a board-edge U.FL, L2 an uninterrupted reference underneath,
stitching vias down both flanks. Stackup L1 signal / L2 ground / L3 power+signal / L4 signal
also relieves the 24 display FPC nets and the boost switch node.

**D4.** Panel carried by the enclosure on standoffs above the front components. Follow-ons:
move J2 from back to front (it currently forces the flex to wrap the board edge for nothing);
verify front component height against the standoff gap in the Phase 4 3D export (RAK811 at
\~1.1 mm is the tallest); enclosure needs a vented-but-sealed path to the SHT45 for IP54.

## 4. Pin budget (verified against RAK811 datasheet Table 4-2)

Free module GPIOs: PA0, PA1, PA8, PA9, PA10, PA12, PA15, PB2, PB4, PB5, PB10, PB11, PB12,
PB14, PB15 — **15 available**. Committed already: PB8/PB9 I2C1, PA13/PA14/PB3 SWD+SWO,
PA2 nPM_PG, BOOT0, RST.

| Function | Pins |
|---|---|
| Shared bit-bang SPI bus (SCK, MOSI) — display + flash | 2 |
| Display control (CS, DC, RST, BUSY) | 4 |
| Flash (CS, MISO) | 2 |
| LIS2DW12 interrupt (I2C shared) | 1 |
| Phototransistor -> ADC-capable pin | 1 |
| PMIC interrupt (nPM GPIO0) | 1 |
| **Subtotal** | **11 of 15** |
| UART debug (PA9/PA10) | 2, kept |
| Spare | 2 |

Sharing SCK/MOSI between display and flash (separate CS lines) buys back the 2 pins that
keep the UART debug header. nPM2100 and LIS2DW12 both join I2C1 rather than taking their own.

### 4.1 Added channels — parts selected 2026-09-23

| Channel | Part | LCSC | Package | Cost | Stock | Temp |
|---|---|---|---|---:|---:|---|
| Shock / tilt | **LIS2DW12TR** (ST genuine) | C189624 | LGA-12 2x2 | $1.19 | 16.9k | -40..+85 |
| Light / door | **Everlight MPT60363T** | C50493 | 0603 | $0.06 | 39.5k | -40..+85 |

- **Use the genuine ST part.** The MSKSEMI and HXY clones on LCSC are 12-bit with 500 nA
  standby versus ST's 16-bit / 50 nA — the low standby current is the whole point here.
- **MPT60363T was chosen specifically for its -40 C rating.** Almost every other stocked
  phototransistor is -25..+85, which would have broken the cold-storage envelope that
  justified this SKU split in the first place.
- **Caveat to validate at bring-up:** the MPT60363T peaks at 940 nm, so it responds strongly
  to daylight (dock doors — the highest-value event) but weakly under LED-only lighting,
  which emits almost no IR. Silicon still gives useful visible response, so lights-on/off
  should register, but confirm on real hardware. Upgrade path if true lux accuracy is wanted:
  **TI OPT3004DTSR** (C5220076, $0.76, -40..+85, 23-bit, I2C, interrupt). Worth considering
  regardless — cumulative light exposure degrades some FMCG goods (dairy, beer), so logged
  lux is a sellable channel, not just a door flag.
- Candidate footprints exist in stock libraries (`Package_LGA:LGA-12_2x2mm_P0.5mm` or the
  Kionix border variant; `LED_SMD:LED_0603_1608Metric`) but **both must be verified against
  the manufacturer land patterns in Phase 3** — the generic LGA-12 may not match ST's pad
  arrangement.

## 5. Execution phases — SKU A

**Phase 0 — Housekeeping. DONE 2026-09-23.**
- Baseline copied to `build/baseline/` (no git commit made — not requested).
- **4-layer stackup applied**: JLC04161H-7628, 1.6 mm — 0.035 / 0.2104 prepreg / 0.0152 /
  1.065 core / 0.0152 / 0.2104 prepreg / 0.035, FR4 er 4.3, ENIG. Inner layers added with
  KiCad 10 numbering (In1.Cu=4, In2.Cu=6, B.Cu=2), verified by SVG export of the inner layers.
- **50 ohm microstrip width = 0.371 mm** (Hammerstad-Jensen, h=0.2104, er=4.3, t=0.035).
  Insensitive: 0.351-0.389 mm across er 4.05-4.6. Guided wavelength at 868 MHz is 191 mm, so
  **lambda/20 = 9.6 mm** — keep the U.FL within ~10 mm of U2 pin 33 and the feed is
  electrically short regardless of exact impedance.
- **Net classes** created: Default (0.2 mm), RF (0.371 mm, 0.30 clearance), Power (0.5 mm),
  Battery (0.8 mm), with netclass patterns for `RF_*`, `ANT*`, `+3V0`, `VOUT_LDO`, `VINT`,
  `VBAT`, `SW`. Design rules set to JLC 4-layer with margin: 0.127 mm min track/clearance,
  0.45 mm min via, 0.25 mm min hole, 0.3 mm copper-to-edge.
- **Project-local libraries** created (`lorawan-airrh.kicad_sym`, `lorawan-airrh.pretty`,
  `sym-lib-table`, `fp-lib-table`); nPM2100 re-pointed from the global SUDOCORP lib.
- **Symbol drift resolved.** Verified the global and cached nPM2100 symbols were
  pin-for-pin and graphically identical — drift was field positions only, no electrical
  risk. Cached entry replaced; **ERC 50 -> 49**, `lib_symbol_mismatch` cleared. Netlist
  confirmed unchanged: 65 nets, zero membership changes.

### Phase 1 — COMPLETE 2026-09-23.  ERC 50 -> 5 (all 5 intentional). 40 -> 54 components.

**Fixes to existing circuitry**
- **B9 footprints (13 parts).** Pulled the GDEY0213B74 reference circuit (sec 12) as an image:
  every display-rail cap is **1uF/25V**, bulk **4.7uF/25V**. Those rails swing to +-20 V, so the
  original 0402 assignments were a field-failure mechanism, not just a wrong land pattern.
  0603/25V on C3-C7, C19-C21; 0805 on C1/C2; rating now carried in the Value field.
  Also confirmed R1/R2 *designators* are transposed vs the datasheet but functions/values are
  correct (R1=2R2 sense, R2=1M gate pulldown) - not a bug, but note it before "fixing" it.
- **B7 RESE.** `DISP_RESE` = J2.3 + Q1.2 + R1.1. Booster control loop closed at the sense
  junction; it was an open loop.
- **B2/B3 GPIO assignment.** Display and flash share SCK/MOSI with separate CS, which bought
  back the 2 pins that keep UART debug. `SPI_SCK` = J2.13+U2.25+U4.6, `SPI_MOSI` =
  J2.14+U2.26+U4.5, plus DISP_CS/DC/RST/BUSY and FLASH_CS/MISO. W25Q16 WP#/HOLD# tied to +3V0.
- **B4/B5 I2C.** Added R7/R8 4.7k pull-ups (there were none at all; nPM2100 requires them).
  PMIC joined the shared bus: `I2C1_SCL` = R8.1, TP5, U1.2, U2.18, U3.7, U5.1.
- **B6 SHPHLD.** SW1 ship/wake button to GND.
- **B1 RF front end.** pi-match (C24/C25 DNP, L3=0R) + J4 U.FL. `RF_OUT` = C24.1+L3.1+U2.33,
  `RF_ANT` = C25.1+J4.1+L3.2.
- **B8 corrected a documentation bug.** The sheet note claimed "VSET Grounded: VOUT = 3.0V".
  Grounding VSET actually gives **1.8 V**; 3.0 V is what the *floating* pin gives. Note now
  states the real constraint: firmware must raise BOOST.VOUT to 3.3 V over I2C before the
  first TX, because VSET cannot select 3.3 V at all.
- **D1 battery.** BT1 swapped to `Device:Battery` + `BatteryHolder_Keystone_2468_2xAAA`,
  value "2xAAA 3.0V". (Battery pin 2 sits 2.54 mm below Battery_Cell's - wire trimmed.)
- PWR_FLAGs added on GND and VBAT; TP6 moved onto grid; unused display pins
  (VPP/TSCL/TSDA/NC) and spare GPIOs no-connect flagged.

**New channels**
- **U5 LIS2DW12TR** (shock/tilt) with C22/C23 decoupling. Symbol built to KLC layout
  (inputs left, outputs right, power top, grounds bottom, NC hidden, alternates for
  SPC/SDI/SDO rather than slash-names) and render-verified. Footprint + STEP model imported
  via easyeda2kicad into the project libraries.
  **Critical finding:** CS and SA0 have ~20.4k *internal* pull-ups to VDD_IO. Tying either
  LOW would sink 3.3V/20.4k = **162 uA continuously = 1418 mAh/yr, more than the entire
  battery**. Both are tied HIGH (CS high = I2C mode, SA0 high = address 0x19). Datasheet also
  mandates RES (pin 7) to GND - done. This is annotated on the sheet.
- **Q2 MPT60363T + R9 1M** (light/door). 1M chosen so worst case in full light is ~3.3 uA;
  ~nA in the dark. Threshold needs hardware validation (940 nm peak: strong on daylight,
  weak under LED-only lighting).
- **TP7/TP8** on PA9/PA10 - UART is the only host interface the module exposes, worth having
  for bring-up.
- **nPM_INT**: PMIC GPIO0 -> U2 PA8.

**I2C address map** (no conflicts): SHT45 0x44, LIS2DW12 0x19, nPM2100.

**Remaining 5 ERC, all intentional and annotated on the sheet:**
- `pin_to_pin` x3 - U3 VINT pins 14/15 are one internal rail bonded out twice; U4 WP#/HOLD#
  are bidirectional pins tied to the +3V0 rail, which is what standard-SPI operation requires.
- `multiple_net_names` x2 - DISP_BS1 on GND and DISP_VCI on +3V0. The descriptive display-pin
  names are deliberately kept for readability; KiCad just reports the net has two names.

**Verification discipline:** netlist exported and diffed after every edit; the cosmetic
cleanup pass was confirmed to leave all 52 nets byte-identical.

**Phase 1 — Schematic completion** (per `kicad-schematic`: iterate on rendered SVGs, validate
deterministically against the exported netlist).
1. Fix the 13 footprint assignments (B9) — first, so BOM and PCB stay honest.
2. Swap BT1 to the 2x AAA holder (D1).
3. Wire display control + flash bus to assigned GPIOs (B2, B3).
4. Add I2C pull-ups to VOUT; join nPM_SDA/SCL to I2C1 (B4, B5).
5. Add the SHPHLD button + series resistor (B6).
6. Connect RESE to the Q1/R1 sense node (B7).
7. Build the RF front end: pi-match + U.FL (B1).
8. Annotate the VSET / rail-sequencing constraint (B8).
9. No-connect flags on all unused module and display pins.
10. Redraw for cleanliness: subcircuit boxes, net colouring by function, power symbols,
    consistent 50 mil labels.

**Phase 2 — ERC to zero.** No errors; warnings only where documented and excluded with a reason.

### Phase 2 & 3 — COMPLETE 2026-09-23.  **ERC = 0.**  56 components, 0 missing footprints.

**ERC driven to zero (was 50):**
- Replaced the W25Q16 /WP + /HOLD hard tie with **10k pull-ups (R10/R11)**. Not ERC
  appeasement: datasheet 4.3/4.4 says those pins *become IO2/IO3 outputs* if the QE bit is
  ever set, so a hard tie to VCC is a driver-contention path. Pull-ups draw 0 uA (pins are
  inputs) and keep quad mode available.
- nPM2100 **VINT pin 15 -> passive** in both the project library and the schematic cache.
  Pins 14/15 are one internal rail bonded out twice; only one can declare itself the source.
  (No jumper-pin-group support in this KiCad build.)
- Built a real **GDEY0213B74_FPC24 symbol** to replace the generic `Conn_01x24_Socket`, with
  the datasheet's actual pin names and types. Identical pin geometry, so the swap was a
  drop-in: **54 nets, zero membership changes.** Power pins then took real power symbols,
  clearing the last 2 `multiple_net_names`. Bonus: unconnected nets now read `J2-VPP-Pad19`
  instead of `J2-Pin_19-Pad19`, and the connector is self-documenting.

**Phase 3 footprint verification — two real errors caught:**

| Part | Finding |
|---|---|
| **U3 nPM2100** | **Was `QFN-16 P0.5mm EP2.45`. Nordic specifies e=0.65 mm, D2/E2=2.65 mm.** Wrong pitch — 4 pins/side span 1.95 mm not 1.5 mm, so it would not have soldered. Now `UQFN-16-1EP_4x4mm_P0.65mm_EP2.6x2.6mm_ThermalVias` (2.6 is the conservative land vs 2.65 nominal). EP is AVSS1 and must stitch to the L2 plane. |
| **U5 LIS2DW12** | easyeda2kicad's generated land was 0.118 mm off terminal centre and 0.23 mm past the package edge. ST's drawing gives 0.275x0.25 terminals at +-0.7625 mm, which **matches KiCad stock `Package_LGA:LGA-12_2x2mm_P0.5mm` exactly** — switched to it (also has a STEP model). Rejected file kept in `build/rejected/`. |

Verified-correct as assigned: SOIC-8 208-mil for W25Q16JVSS (D=E=5.28, e=1.27);
`RF_Module:RAK811` (34 pads, pad 33 RF_OUT on the top edge flanked by GND 32/34);
SHT4x **NoCentralPad** — Sensirion explicitly says *not* to solder the die pad, which also
helps the thermal isolation this product depends on.

New footprint built from the manufacturer land pattern (no KiCad stock part):
`lorawan-airrh:Texas_DTS0008A_SOT-5X3-8` — TI drawing 4226132/G, 0.72x0.3 mm pads at
+-0.86 mm, 0.5 mm pitch. easyeda2kicad's version was undersized (0.68x0.28 at +-0.79).

**Light channel upgraded** (the -40 C requirement was the only reason for the bare
phototransistor): Q2/R9 replaced by **U6 OPT3004DTSR** — 23-bit true lux over I2C with a
threshold interrupt. **ADDR is tied HIGH for 0x45; ADDR low would be 0x44, the SHT45's
address.** Needs a clear optical path — must not sit under the display.

**SW1** resolved to **XKB TS-1187A** (JLC Basic C318884, 5.1x5.1 mm, 1.5 mm tall); KiCad
stock footprint exists and its pads are numbered 1,1,2,2 so it maps onto the 2-pin symbol.

**Still open for Phase 3:** confirm the J2 FPC contact side against a physical sample or
Good Display's DESPI-C02 adapter. Bottom-contact is the common pairing and is what is
assigned, but it is unconfirmed. Low layout risk — top- and bottom-contact variants in this
family usually share a land pattern, so it is a BOM swap rather than a respin.

**Phase 3 — Footprints + 3D.** Verify every footprint against its datasheet land pattern
(`kicad-footprint`). Attach 3D models for U2, U3, J2, BT1 and the display panel so the D4
mechanical stack can be checked visually.

**Phase 4 — Layout** (`kicad-layout` / `kicad-pcb`, editing
`lorawan-airrh/lorawan-airrh.kicad_pcb` in place, backups to `build/`).
1. Push the netlist to the board.
2. Re-place: RAK811 RF corner at a board edge with a short antenna feed; PMIC loop
   (BT1-C11-L1-U3-C13) tight; **SHT45 thermally isolated** — slot/cutout, copper kept away,
   since self-heating lands directly on the headline spec; display booster grouped near J2;
   AAA holder on the back clear of the module's RF corner.
3. Ground pour, via stitching, RF keepout under the feed.
4. Mounting holes, fiducials, chamfered outline.
5. **Stop. No routing** — hand off to the engineer (see standing constraint above).
   Deliverable is a placed, netlist-synced board with zones and keepouts defined and
   ratsnest visible. Note DRC will legitimately report unconnected items at handoff.

### Phase 4 — PLACEMENT HANDOFF 2026-09-24.  ERC 0, DRC 0 errors, parity 0.  Not routed.
Placement is the engineer's. Agent pass only added the finishing items below
(backups: `build/backups/*.user-placed`, `*.pre-rules`, `*.pre-zones`).
- **Net names:** `VBAT`, `BOOST_SW`, `VINT`, `DISP_SW`, `DISP_PUMP` labelled in the schematic
  (were auto `Net-(...)`, so the old net-class patterns never matched). Board renamed to match.
- **Net classes:** Default 0.15/0.15, via 0.45/0.2 · Power 0.4 (`+3V0 VOUT_LDO /VINT /BOOST_SW`)
  · Battery 0.6 (`/VBAT`) · RF 0.371 w / 0.20 gap, CPWG ~50 ohm over In1 GND (`/RF_*`)
  · Display_HV 0.2/0.2 (`/DISP_V* /DISP_SW /DISP_PUMP /DISP_GDR`, +/-20 V rails).
- **Board minimums = JLC 4-layer limits:** 0.09 track/space, via 0.15/0.25, annular 0.05,
  hole clearance 0.2, hole-to-hole 0.2, edge 0.2, mask web 0.1, silk 0.15/1.0 mm text.
  PTH-specific limits, the >=0.2 mm via-cost warning, the RF and HV rules and "no blind/micro
  vias" live in `lorawan-airrh.kicad_dru`. All DRU values need explicit units (`0.2mm`); a DRU that
  fails to parse is silently dropped by kicad-cli. Fixed 2026-09-24; kicad-cli now applies it.
- **Zones:** stackup SIG / GND / GND / SIG (decided 2026-09-24; power routed as
  traces): GND planes on In1.Cu (RF reference) and In2.Cu; pour keepout (all copper) over the SHT45
  thermal island. Filled.
- **Outline:** two 33 nm slivers at the J2 notch reliefs and a 25 um bottom-edge step removed.
- **Mechanical note:** assumed panel outline drawn on User.1 (29.2 x 59.2, X-centred,
  FPC edge at board bottom). TP7/TP8 nudged 1.55 mm outboard to clear the panel edge.
- **Open for the engineer:** J3 Tag-Connect sits under the panel edge (program before the
  display goes on, or move it to the top strip); H1 screw head under the panel corner;
  L2 1210 wirewound ~2.2 mm tall against a 2-3 mm standoff (Murata LQH3NPN470MMEL,
  3x3x~1.1 mm, 380 mA Isat, is the low-profile swap); 4 older 0402 library copies
  (C8-C10, R5) — Update Footprints from Library clears them; 8 refdes silk overlaps.

**Phase 5 — Outputs.** DRC to zero (`kicad-export`). Gerber render review (`kicad-gerbers`).
BOM with MPN/LCSC fields (`kicad-bom`). Assembly drawings, 3D export, product render.

## 6. SKU B — forward plan (not started)

Fork from SKU A **after** it is fabbed and validated, so the shared blocks (MCU + radio + RF,
display, flash, SHT45) are frozen and proven before they are copied. Deliberately *not*
restructuring SKU A into cross-project hierarchical sheets now: at ~40 parts a flat sheet
with boxed subcircuits is what `kicad-schematic` calls for, and premature hierarchy costs
more than it saves while SKU B is this far out. Revisit if the two boards end up churning in
parallel.

Changes from SKU A: mains/external DC power section replacing the nPM2100 + AAA; STCC4 CO2;
SGP41 VOC/NOx; TGS5141 + LMP91000 CO front end; I2C bus isolation for any switched sensor
rail; outline grown to ~46 x 88 mm with a vented gas chamber; ePTFE vent membrane for IP54.

Notes carried forward for that work:
- **STCC4** — use `set_rht_compensation` (0xE000) fed from the SHT45 rather than the
  datasheet's dedicated second SHT4x; solder SDA_C/SCL_C to floating pads. Accuracy is
  specified at 10 s sampling; ASC needs ~360 measurements to converge after power-off and
  assumes weekly exposure to ~400 ppm fresh air.
- **TGS5141** — 1.2-3.2 nA/ppm, so 50 ppm gives 60-160 nA. Through the LMP91000's max
  350 kOhm TIA that is 56 mV, ~70 LSB on the STM32 12-bit ADC (~0.7 ppm/LSB). Noise-limited:
  oversample, and evaluate an external higher-gain TIA at bring-up. Cell voltage must stay
  within +-10 mV, which is what the potentiostat guarantees.
- **I2C addressing** — SHT45 0x44, LMP91000 0x48, SGP41 0x59, STCC4 0x64, nPM2100 separate.
  No conflicts; size pull-ups for five devices' bus capacitance.
- **SGP41 is ruled out.** Confirmed 2026-09-23: SKU B requires VOC in **ppb**, and SGP41 —
  like every MOX part — outputs only an unitless Gas Index. Real ppb TVOC means
  **photoionization (PID)**: ION Science MiniPID 2, Alphasense PID-AH2 or similar. Those
  cells run roughly $150-400 and tens of mA, which moves SKU B into a materially higher cost
  and power band than the earlier estimate in this plan. Re-price SKU B before committing.
  (SGP41 reference figures, if a MOX fallback is ever reconsidered: 3.0 mA typ continuous at
  3.3 V, 34 uA idle, Gas Index Algorithm needs continuous 0.5-10 s sampling.)
