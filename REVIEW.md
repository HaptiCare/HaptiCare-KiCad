# HaptiCare PCB – Design review & path to JLCPCB order

Review date: 2026-11-10 · Target: order from JLCPCB (assembled, 4-layer) by ~17 Nov 2026 so boards arrive within 3 weeks.

## Status

- **Schematic:** complete (single flat sheet). This PR adds LCSC part numbers and a few hardware changes (see below).
- **PCB:** 70 × 54.65 mm, 4 layers (In1 = GND plane, In2 = split 3V3/5V). All parts placed, ~95 % routed.
- **Copper check:** no clearance violations, no courtyard overlaps, antenna keep-out clean, all pinouts match the datasheets.
- **KiCad DRC (2026-11-10):** 11 violations and 3 unconnected items. None needs a reroute; details are in the DRC section below.
- **Needs work:** 3 unrouted connections, mic footprint mask fix, the IP5306 switching loop, power trace widths, and placing the new parts from this PR.

## Changes in this PR (schematic only)

| Change | Details |
|---|---|
| LCSC field on every BOM part | Except J6/J9 (hand-soldered sockets) and C15 (DNP) |
| L1 swapped | Coilcraft XAL4020 (out of stock at LCSC) → **XFL4020-102MEC** (C5355380). Same `L_Coilcraft_XxL4020` footprint, so no layout change |
| J5 battery connector | 2.54 mm pin socket → **JST PH 2-pin** B2B-PH-K (C131337). Pin 1 = GND, pin 2 = BAT. **Check the battery's polarity**, it varies by vendor |
| C15 | Marked DNP |
| **New:** IP5306 keep-alive | Q5 (AO3400A) in parallel with SW1 (net `KEY_BTN`). Gate driven from **IO5** (`KEEPALIVE`) via R22 100 Ω; R23 10 k pull-down |
| **New:** battery voltage sense | R24/R25 100 k / 100 k divider BAT → **IO4** (`BAT_SENSE`, ADC1_CH3), C21 100 nF filter |
| IO4 / IO5 | No-connect flags removed |

### Why the keep-alive

The IP5306 switches its 5 V output off when the load stays below ~45–50 mA for ~32 s. This board idles at ~40–45 mA (ESP32 + mics), right at that threshold.

**Firmware rule:** pulse IO5 high for ~100 ms every ~20 s. Never send two pulses within ~1 s, because a double press toggles the IP5306 flashlight function. Keeping BLE/Wi-Fi or the display active also keeps the load above the threshold.

The IP5306-I2C variant was considered but not chosen: its "always-on boost" register is documented only by third parties (M5Stack and community drivers), not in the public datasheet.

## Tasks before ordering

### 1. Electrical / layout (must do)

- [ ] **Update PCB from Schematic** and place Q5, R22, R23, R24, R25, C21 and the new J5 footprint
- [ ] Route **USB D−** (U1 pin 13 ↔ U3 pin 6). Without it, native USB programming does not work
- [ ] Connect **C6 pad 2 (GND)**: the GND track ending at (137.94, 120.05) does not reach the pad. Extend it or add a via to the plane
- [ ] Close **Reset1** second pad 2: (153.5, 110.75) ↔ (161.5, 110.75). Probably connected inside the switch, but DRC should be clean
- [ ] **IP5306 switching loop:** place L1 directly at IC1 pin 7 (SW). Short, wide trace, top layer only, no vias. C7/C9 close to the IC
- [ ] Widen **BAT, +5V, +Vbus** from 0.4 mm to **≥ 1 mm** (or pours). Charge and boost currents reach ~2 A, motor start-up adds spikes
- [ ] Add 4–6 thermal vias under the IP5306 exposed pad (pin 9)
- [ ] DRC: **0 errors, 0 unconnected items** (see the DRC findings below)

### DRC run 2026-11-10 (11 violations, 3 unconnected)

| DRC finding | Count | Cause | Fix | Reroute? |
|---|---|---|---|---|
| Unconnected: USB D− | 1 | Gap between the track at (153.52, 114.07) and the track at (142.70, 114.20) | Close the gap | Short track only |
| Unconnected: C6 GND | 1 | GND track stops short of C6 pad 2 | Extend the track or add a via | No |
| Unconnected: Reset1 pad 2 | 1 | The two pad-2 pads of the switch are not joined by copper | Short track between them | No |
| Solder mask bridge: MK3, MK4 | 8 | ICS-43434 footprint pads (0.6 × 0.522 mm, 0.82/0.9 mm pitch) have a 0.102 mm mask margin each. That leaves a ~0.094 mm mask web, below the 0.1 mm board minimum | In the footprint editor, set the mask margin of pads 1, 2, 4, 5, 6 to **0.05 mm** (web becomes ~0.2 mm), then update both mics from the library. Do not just allow bridges: on an LGA part the web prevents solder bridges | No |
| Copper-to-edge clearance | 3 | Inner zones (GND In1, +3V3/+5V In2) are drawn beyond the board outline, so the fill is clipped to 0.4973 mm instead of 0.5 mm (arc rounding) | Board Setup → Constraints: copper-to-edge 0.45 mm (JLCPCB needs ≥ 0.3 mm), or redraw the zone outlines inside the board | No |

MK4 pad 3 (GND) does **not** appear in the KiCad DRC, so it is connected. My earlier script check had approximated the custom pad shape. The new parts from this PR (Q5, R22–R25, C21, new J5) are not in the PCB yet: run *Update PCB from Schematic* first, then re-run DRC.

### 2. BOM

- [ ] Optional: put the capacitor voltage into the value (e.g. `22uF 25V`). The chosen parts are 25–50 V
- [ ] Exclude J6/J9 from assembly (hand-solder)

### 3. Fabrication files

- [ ] Export Gerbers + drill, BOM and CPL (placement file) in JLCPCB format. Use a KiCad JLCPCB plugin if it works with KiCad 10, otherwise export manually
- [ ] Upload to JLCPCB and **check the 3D assembly preview**. Rotations are the classic mistake: ESP32 module, USB-C, mics, diodes, LEDs
- [ ] Check stock at order time. Expect JLCPCB's setup fee of ~3 USD per extended part (ESP32 module, IP5306, mics, USB-C, inductor, buttons, USBLC6)
- [ ] Second-person review, then order ~5 assembled boards

### 4. In parallel

- [ ] Order batteries (with JST PH plug), coin vibration motors, OLED (SSD1306 0.96" I2C) and JST PH cables

## Not in scope for this revision

- Merging both mics onto one stereo I2S bus (L/R pins already set opposite, would free 2 GPIOs)
- Deleting the orphaned sheets (`Microphone/Motors/OLED/ESP32/power.kicad_sch`)
- The stray SOT-23-5 footprint on the PWR_FLAG symbols (harmless)
- Sleep-mode support (not needed for now)

## Estimated load

Assumes an SSD1306 OLED and coin vibration motors used only for short haptic pulses.

| Mode | Battery current, no display | Battery current, with display |
|---|---|---|
| Audio processing, radio off | ~70 mA | ~95 mA |
| BLE active | ~140 mA | ~165 mA |
| Wi-Fi | ~220 mA | ~250 mA |

- Motor pulses add only ~1–3 mA on average. Peaks reach ~1 A at 5 V if several motors start during a Wi-Fi burst.
- ME6211 regulator: Wi-Fi peaks ~350 mA plus display, within its 500 mA rating.
- Rough battery life on 1000 mAh: ~6–7 h with BLE, ~10–14 h audio-only.

## BOM with LCSC numbers

Stock read from LCSC on 2026-11-10; "basic" = JLCPCB basic part (stock not checked individually).

| Ref | Value | LCSC | Manufacturer part | Stock | Note |
|---|---|---|---|---|---|
| U1 | ESP32-S3-WROOM-1 | C2913201 | Espressif ESP32-S3-WROOM-1-N8R8 | 7,082 | IO35–37 must stay free (they are) |
| IC1 | IP5306 | C181692 | Injoinic IP5306 | 23,565 | |
| MK3, MK4 | ICS-43434 | C5656610 | TDK InvenSense ICS-43434 | 4,437 | ~2.8 USD each |
| U2 | ME6211C33M5 | C82942 | Microne ME6211C33M5G-N | basic | |
| U3 | USBLC6-2P6 | C15999 | ST USBLC6-2P6 | 18,230 | SOT-666 |
| Q1–Q5 | AO3400A | C20917 | AOS AO3400A | basic | Q5 = keep-alive |
| D1–D4 | SS14 | C2480 | MDD SS14 | basic | |
| D5–D9 | LED | C84256 | NCD0805R1 (red) | basic | |
| L1 | 1 µH | C5355380 | Coilcraft XFL4020-102MEC | 10,949 | 5.4 A Isat, 10.8 mΩ |
| J11 | USB-C 16P | C3020560 | GCT USB4105-GF-A | 16,238 | |
| J1–J5 | JST PH 2-pin | C131337 | JST B2B-PH-K-S(LF)(SN) | not checked | J1–J4 motors, J5 battery |
| SW1, Boot1, Reset1 | SW_Push | C271750 | Omron B3FS-1000P | 68,220 | |
| C3–C5, C12, C17–C20 | 22 µF | C45783 | Samsung CL21A226MAQNNNE | basic | 25 V X5R |
| C6–C9 | 10 µF | C15850 | Samsung CL21A106KAYNNNE | basic | 25 V X5R |
| C10, C11, C14 | 1 µF | C28323 | Samsung CL21B105KBFNNNE | basic | 50 V X7R |
| C1, C2, C13, C21 | 100 nF | C49678 | Yageo CC0805KRX7R9BB104 | basic | 50 V X7R |
| C15 | NC | – | – | – | DNP |
| R2, R4, R6, R8, R10, R15, R17, R23 | 10 kΩ | C17414 | UNI-ROYAL 0805W8F1002T5E | basic | |
| R18, R19, R24, R25 | 100 kΩ | C149504 | UNI-ROYAL 0805W8F1003T5E | basic | |
| R20, R21 | 4.7 kΩ | C17673 | UNI-ROYAL 0805W8F4701T5E | basic | |
| R13, R14 | 5.1 kΩ | C27834 | UNI-ROYAL 0805W8F5101T5E | basic | |
| R3, R5, R7, R9, R22 | 100 Ω | C17408 | UNI-ROYAL 0805W8F1000T5E | basic | |
| R12 | 20 Ω | C17544 | UNI-ROYAL 0805W8F200JT5E | basic | |
| R1, R11 | 2 Ω | C17606 | UNI-ROYAL 0805W8F200KT5E | basic | |
| R16 | 0 Ω | C17477 | UNI-ROYAL 0805W8F0000T5E | basic | |
| J6, J9 | 2.54 mm sockets 1×4 / 1×6 | – | generic | – | Hand-solder |
