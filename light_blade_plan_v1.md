# Light Blade – PCB plan v1

A simple daisy-chainable LED board for the Elite 4D-EXMU controller. Each board carries:

- **10 × 0805 single-colour SMD LEDs**, each one individually controllable.
- **6 × 3 V 38 mm LED filaments**, each one individually controllable.

Every board is driven by WS2811 single-wire (WS281x protocol) driver chips, so the controller treats the chain like ordinary pixels.

KiCad project: `light_blade/light_blade.kicad_sch` (KiCad 10)

---

## 1. Design decisions

| Topic | Decision | Why |
|---|---|---|
| Driver IC | **WS2811, SOP-8** | It has 3 constant-current outputs (18.5 mA each) and speaks the standard WS281x protocol. |
| Supply | **5 V** | At 12 V each chip would burn about 0.5 W driving single 2–3 V LEDs. The Elite passes the PSU voltage straight through. |
| Speed | **SET pin left open → 800 kHz** | From the WS2811 datasheet: SET tied to VDD = 400 kHz, SET floating = 800 kHz. |
| VDD feed | **100 Ω in series with VDD, plus 100 nF to GND, on every chip** | This is the datasheet's 5 V reference circuit ("not more than 100 Ω"). |
| Data line | **33 Ω in series at every DIN, and one more before the OUT connector** | The datasheet recommends this for reflections and hot-plug protection. |
| Small LEDs | **0805**, connected from 5 V to one output, no resistor | The output is a constant-current sink. 0805 handles 18.5 mA more comfortably than 0603. |
| Filaments | **All 3 outputs of one WS2811 tied together → ~55 mA per filament** | A filament typically wants 50–100 mA, so a single 18.5 mA output would leave it dim. Check the AliExpress listing for the real figure. |
| Filament assembly | **Hand-soldered**, marked DNP | They are loose parts with solder tabs, not SMD reels, so PCBWay cannot place them. |
| Connectors | **JST XH, 3-pin, right-angle (side entry)**, one for IN and one for OUT | Cheap, rated 3 A, and PCBWay can source them. Note the pitch is actually **2.50 mm**. |

### Chip allocation

| Chip | Drives | Pixel # in controller |
|---|---|---|
| U1 | D1 (R), D2 (G), D3 (B) | 1 |
| U2 | D4, D5, D6 | 2 |
| U3 | D7, D8, D9 | 3 |
| U4 | D10 (R), D17 (G), D18 (B) | 4 |
| U5 … U10 | Filaments D11 … D16, one per chip, R/G/B paralleled | 5 … 10 |

In the Elite/WLED controller, each board is **10 RGB pixels**:
- Pixels 1–4 control the small LEDs through their R/G/B channels.
- For pixels 5–10 (the filaments), use **white (R=G=B)** or any equal grey level. The three paralleled channels add together, so unequal values just give an in-between brightness.

---

## 2. Bill of materials (per board)

| Ref | Qty | Part | Value | Footprint | Assembly |
|---|---|---|---|---|---|
| U1–U10 | 10 | WS2811 SOP-8 | WS2811 | `Package_SO:SOIC-8_3.9x4.9mm_P1.27mm` | PCBWay |
| D1–D10, D17, D18 | 12 | 0805 LED (KENTO, Everlight, … from LCSC) | LED 0805 white | `LED_SMD:LED_0805_2012Metric` | PCBWay |
| D11–D16 | 6 | LED filament 3 V 38 mm (AliExpress) | – | `light_blade:LED_Filament_38mm` (custom) | **Hand-solder (DNP)** |
| R1–R10 | 10 | Resistor 0805 | 33 Ω | `Resistor_SMD:R_0805_2012Metric` | PCBWay |
| R11–R20 | 10 | Resistor 0805 | 100 Ω | `Resistor_SMD:R_0805_2012Metric` | PCBWay |
| R21 | 1 | Resistor 0805 | 33 Ω | `Resistor_SMD:R_0805_2012Metric` | PCBWay |
| C1–C10 | 10 | Ceramic capacitor 0805, X7R | 100 nF | `Capacitor_SMD:C_0805_2012Metric` | PCBWay |
| C11 | 1 | Electrolytic SMD capacitor | 470 µF / 10 V | `Capacitor_SMD:CP_Elec_8x10` | PCBWay |
| J1, J2 | 2 | JST XH 3-pin right-angle: S3B-XH-A (genuine) or "XH-3AW" clone | IN / OUT | `Connector_JST:JST_XH_S3B-XH-A_1x03_P2.50mm_Horizontal` | PCBWay |

**Cables (bought separately):**

- Board to board: **"JST XH 2.54 mm 3-pin cable, double-ended (female–female)"**, 5–20 cm.
- Controller to first board: **"JST XH 3-pin pigtail, single-ended"**. The bare wire end goes into the Elite's screw terminals.

**Connector pinout (same on IN and OUT):** pin 1 = +5V, pin 2 = DATA, pin 3 = GND.

### Power budget

- 10 LEDs × 18.5 mA ≈ 185 mA
- 6 filaments × ~55 mA ≈ 330 mA
- Chips ≈ 10 × ~3 mA ≈ 30 mA

**Total ≈ 0.55 A per board at full white.** The XH contacts are rated 3 A, so the 5 V rail can feed about **5 boards in series**. For longer chains, inject 5 V partway along.

---

## 3. The generated schematic

`light_blade/` contains:

| File | Content |
|---|---|
| `light_blade.kicad_pro` | Project file (open this one in KiCad) |
| `light_blade.kicad_sch` | Complete, wired schematic |
| `light_blade.pretty/LED_Filament_38mm.kicad_mod` | Custom filament footprint |
| `fp-lib-table` | Registers the `light_blade` footprint library for this project |

All footprints are already assigned. KiCad's netlist export confirms that every pin is connected as intended:
- 29 nodes on +5V and 23 on GND.
- DATA0 → DATA10 → DATA_OUT runs chip to chip.
- Each FILx_K net joins one chip's 3 outputs to its filament.

**Expected ERC result: 12 errors, "Pins of type Output and Output are connected."** These are the deliberately paralleled outputs of U5–U10. In the ERC dialog, right-click each one → **Exclude this violation**, so later runs stay clean.

### Filament footprint

The footprint has two large SMD pads, 4 × 3 mm, **36 mm apart centre to centre**, so the filament lies flat on the board.
- Pad 1 = cathode (−), pad 2 = anode (+).
- The silkscreen marks the ends with − and +.

**Measure your actual filaments before ordering** and adjust the pad spacing in the footprint editor if needed. Filaments usually mark the + end with a small hole or notch in the tab.

---

## 4. Step-by-step: doing it yourself in the KiCad schematic editor

Use this to understand the generated schematic or to rebuild it from scratch.

1. **Create the project.** File → New Project → `light_blade`. Open the schematic.
2. **Set up the page.** File → Page Settings: A3, title "Light Blade", rev v1.
3. **Place the connectors.**
   - Press `A`, search `Conn_01x03`, and place J1 and J2.
   - Set their Value fields to `IN` and `OUT`.
   - Pin 1 → +5V, pin 2 → DATA, pin 3 → GND.
4. **Add power symbols.** Press `P` and place `+5V` and `GND` on connector pins 1 and 3.
5. **Add the bulk capacitor and flags.**
   - Place C11 (`C_Polarized`, 470 µF) between +5V and GND.
   - Add one `PWR_FLAG` on +5V and one on GND. Power arrives through a connector, so ERC needs to be told the net is driven.
6. **Build one driver block** (`Driver_LED:WS2811`):
   - **VDD (pin 8):** connect to +5V through 100 Ω. From the VDD node, 100 nF to GND. Also put a `PWR_FLAG` on the VDD node, because it sits behind the resistor.
   - **GND (pin 4):** to GND.
   - **SET (pin 7):** leave open and place a no-connect flag (`Q`). Open = 800 kHz.
   - **DIN (pin 6):** through 33 Ω to the incoming data label.
   - **DO (pin 5):** to the outgoing data label.
7. **Chain the data line** with net labels (`L`): `DATA0` at J1 pin 2 → U1 … `DATA10` from U10 DO → R21 (33 Ω) → `DATA_OUT` → J2 pin 2.
8. **Copy the block.** Select the finished block and press `Ctrl+D` to duplicate it 9 times. Renumber the data labels.
9. **Wire the small LEDs** (`Device:LED`). Anode to +5V, cathode to a label that matches a driver output (`U1_R`, `U1_G`, …). No resistor.
10. **Wire the filaments.**
    - On each of U5–U10, give OUTR, OUTG and OUTB the **same** label (`FIL1_K` … `FIL6_K`).
    - Filament anode to +5V, cathode to that label.
    - Set the filament symbols to *Do not populate*.
11. **Wire U4's outputs.** OUTR, OUTG and OUTB drive D10, D17 and D18 (labels `U4_R`, `U4_G`, `U4_B`).
12. **Annotate and assign footprints.** Tools → Annotate Schematic, then Tools → Assign Footprints. Use the footprints from the BOM table.
13. **Add sourcing fields.** Give each part an `LCSC` or `MPN` field with the part number for PCBWay's BOM import.
14. **Run ERC.** Inspect → Electrical Rules Checker. Exclude the 12 intentional output-to-output errors described above.

---

## 5. Initial PCB layout (`light_blade/light_blade.kicad_pcb`)

**Shape:** 360 × 50 mm, 2 layers, shaped like a sword.
- **Hilt** (x 0–114): J1, the bulk capacitor C11, LED drivers U1–U4, and the 10 LEDs in a row.
- **Guard:** flared with 45° steps.
- **Blade:** the 6 filaments lie at 45° as slashes, each with its own driver (U5–U10) in the corner beside it. The bottom edge has 45° V-notches between the filaments.
- **Tip:** 45° bevel, with J2 at the point.
- J1 and J2 overhang the board edge by about 1.5 mm.

**Copper:**
- **Top:** +5V copper fill (zone), plus short traces from each driver to its loads. All bends are 45°.
- **Bottom:** solid GND copper fill. The data chain runs as bottom-layer hops along the top edge, chip to chip, in schematic order from left to right.

**DRC (KiCad 10):**
- 0 errors, 0 unconnected items, 0 schematic-parity issues.
- 10 "silkscreen clipped by board edge" warnings, all from J1/J2. They overhang on purpose.

## 6. Next steps (PCB layout)

1. Open `light_blade.kicad_pcb`, review placement and press `B` to refill zones after any edit.
2. Draw the board outline. Put **J1 and J2 on opposite edges, slightly overhanging**, so boards can sit flush side by side. Label them **IN** and **OUT** on the silkscreen.
3. Place each 100 nF capacitor and 100 Ω resistor right next to its chip's VDD pin.
4. Use **≥ 1 mm traces for +5V** (it carries the current for the whole chain) and a **GND pour** on the bottom layer.
5. Lay out the filament pads where they look good. Keep LEDs and filaments away from the connector edges.
6. Run DRC, then export Gerbers, BOM and pick-and-place (CPL) for PCBWay. Leave D11–D16 out of the assembly order.

## 7. Open points to verify before ordering

- [ ] Filament current and tab spacing: measure, or read the listing, then adjust the footprint pad pitch.
- [ ] Filament polarity marking.
- [ ] Exact LCSC part numbers for the LED colour, WS2811, and XH connector (PCBWay stock).
- [ ] Elite 4D-EXMU output voltage and data level on the chosen port (should be 5 V).
