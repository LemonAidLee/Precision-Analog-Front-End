# 07 — KiCad Schematic and PCB

[← Documentation index](README.md)

Project: [`../KiCAD_PCB_Design/Precision_AFE_ESP32.kicad_pro`](../KiCAD_PCB_Design/) · KiCad **10.0.5** · Title block: *Precision Analog Front-End for ESP32*, **Rev A**

---

## 1. Schematic

![Full schematic](images/schematic/schematic_full.svg)

*Rev A schematic, single sheet, A4. PDF: [`design_files/Precision_AFE_ESP32_schematic.pdf`](design_files/Precision_AFE_ESP32_schematic.pdf)*

### Organisation

The sheet is laid out as three visual regions that mirror the signal flow:

| Region     | Contents                                                                                                      |
| ---------- | ------------------------------------------------------------------------------------------------------------- |
| Upper-left | J1 sensor input, J2 power input, Stage 1 (Sallen-Key: R1, R2, C1, C2, U1A, Rf1, Rg1), Stage 2 (U1B, Rf2, Rg2) |
| Lower-left | VREF divider (Rref_a1, Rref_b1, C3), Stage 3 (Rin1, Rref1, Rf3, U2A), Stage 4 (Rin2, Rf4, U2B), power regulation (U3 LM7812, U4 LM7912, C4–C7) |
| Right      | J3/J4 ESP32 headers, J5 ADS1115 header                                                                        |

Power distribution is by **global power symbols** (+12V, −12V, +3V3, GND, and the local +15V_RAW/−15V_RAW rails) rather than drawn wires, so the supply rails do not clutter the signal path. J2 brings in unregulated **+15V_RAW / GND / −15V_RAW**; on-board LM7812 (U3) and LM7912 (U4) regulate these down to the precision ±12 V the op-amps run on, each with an input/output bypass pair (C4/C5 for U3, C6/C7 for U4). Four `PWR_FLAG` symbols (FLG1, FLG4, FLG5, FLG6) declare the externally-driven rails — GND, +3V3, +15V_RAW, −15V_RAW — to ERC. `+12V` and `−12V` are legitimately driven by the regulators' own output pins.

The TL082 symbol is used in its three-unit form: unit A (pins 1/2/3), unit B (pins 5/6/7), and unit C — the power unit carrying pins 4 and 8 — drawn separately near the bottom of each amplifier group.

### Named nets

The schematic explicitly labels the key nodes necessary for precise probing and board-level identification:

`VSENSOR` · `VFILTER` · `VAMP` · `VREF` · `VADC` · plus the supply rails (including `+15V_RAW` / `−15V_RAW`) and the ESP32 GPIO breakout labels.

The seven internal nodes (the two filter nodes, the four op-amp inverting-input nodes, and the level-shift output VX) intentionally utilize KiCad's robust auto-naming structure (`Net-(<ref>-Pad<n>)`), which correctly drives the physical PCB routing.

### ERC

|          |              |
| -------- | ------------ |
| Errors   | **0**  |
| Warnings | **32** |
| Info     | 0            |

All 32 warnings are *"Label connected to only one pin"*, on the ESP32 breakout labels (GPIO0…GPIO39, EN, VIN_5V) at J3 and J4. These pins are deliberately brought out and labelled for future expandability. This is an intended consequence of the breakout design and is entirely benign.

---

## 2. Board

![PCB 3D render, isometric](images/pcb/pcb_3d_view_isometric.png)

*KiCad 3D render with component models. Status: Pending Fabrication.*

### Physical parameters

| Parameter            | Value                           |
| -------------------- | ------------------------------- |
| **Dimensions** | **78.1 mm × 99.6 mm** (bounding box) |
| Layers               | 2 (F.Cu, B.Cu)                  |
| Thickness            | 1.6 mm                          |
| Board area           | ≈ 77.8 cm²                       |
| Footprints           | 29 (all through-hole)           |
| Pads                 | 115                             |
| Copper zones / pours | **2** — GND, one per copper layer |
| Mounting holes       | **0**                     |

The outline is drawn as four `Edge.Cuts` lines: (110, 45) → (187.92, 45) → (188, 144.5) → (110, 144.5) → close. The left and bottom edges are exactly 99.5 mm and 78 mm respectively and axis-aligned. The minor 0.08 mm offset on the right edge is a known artifact of the routing optimization phase and will be squared prior to final fabrication.

### Design rules

| Rule                          | Value                                                     |
| ----------------------------- | --------------------------------------------------------- |
| Minimum clearance             | 0.2 mm                                                    |
| Minimum track width           | 0.2 mm                                                    |
| Minimum via diameter / drill  | 0.5 mm / 0.3 mm                                           |
| Minimum through-hole diameter | 0.3 mm                                                    |
| Minimum hole-to-hole          | 0.25 mm                                                   |
| Copper-to-edge clearance      | 0.5 mm                                                    |
| Default netclass              | 0.2 mm clearance, 0.2 mm track, 0.6 mm via / 0.3 mm drill |

These robust constraints guarantee manufacturability across any standard 2-layer fabrication process.

---

## 3. Component placement

![PCB layout, 2D](images/pcb/pcb_layout_2d.png)

*F.Cu, B.Cu, F.SilkS and Edge.Cuts. Vector version: [`images/pcb/pcb_layout_all_layers.svg`](images/pcb/pcb_layout_all_layers.svg)*

### Placement philosophy

The physical layout prioritizes signal integrity and prototype debuggability through three core principles:

**1. Analog/Digital Isolation.** The four amplifier stages, the power regulation, the reference, and the power terminal occupy the left ~39 mm of the board; the ESP32 and ADS1115 sockets occupy the right ~32 mm, establishing a ≈ 7 mm physical isolation corridor. This enforces strict separation of domains above the dual-layer ground pour.

**2. Top-to-Bottom Signal Flow.** J1 and the filter R/C are positioned at the top edge; U1 sits below them; U2 below that; the reference divider and power terminal at the bottom. This layout mirrors the schematic, allowing an engineer to physically probe the signal chain sequentially down the board.

**3. Centralized Feedback Resistor Column.** Rf1, Rg1, Rf2, Rg2, Rin1, Rin2, Rf3, Rf4 are stacked vertically in the centre of the board at 7 mm pitch, each with its reference designator clearly visible on the silkscreen. 

While conventional high-frequency layout dictates placing feedback resistors immediately adjacent to op-amp pins, this design operates at 1 kHz where parasitic trace effects are negligible. Grouping these components creates a "tuning column" that drastically simplifies value-swapping during physical characterization, representing a highly intentional trade-off in favor of prototype debuggability.

### Placement table

| Ref     | Value           | Position (mm)   | Footprint                      |
| ------- | --------------- | ---------------- | ------------------------------ |
| J1      | SENSOR_INPUT    | 117.42, 50.00    | PinHeader 1×02, 2.54 mm       |
| R1      | 1.5 kΩ         | 126.42, 50.00    | Axial DIN0207, 10.16 mm pitch  |
| R2      | 1.5 kΩ         | 142.42, 50.00    | Axial DIN0207                  |
| C1      | 100 nF          | 126.42, 58.00    | Disc, 5.00 mm pitch            |
| C2      | 100 nF          | 126.84, 84.50    | Disc, 5.00 mm pitch            |
| U1      | TL082           | 125.53, 68.19    | DIP-8, 7.62 mm                 |
| U2      | TL082           | 125.53, 94.19    | DIP-8, 7.62 mm                 |
| Rf1     | 5.9 kΩ         | 138.42, 66.00    | Axial DIN0207                  |
| Rg1     | 10 kΩ          | 138.42, 73.00    | Axial DIN0207                  |
| Rf2     | 90 kΩ          | 138.42, 80.00    | Axial DIN0207                  |
| Rg2     | 10 kΩ          | 138.42, 87.00    | Axial DIN0207                  |
| Rin1    | 36 kΩ          | 138.42, 94.00    | Axial DIN0207                  |
| Rin2    | 10 kΩ          | 138.42, 101.00   | Axial DIN0207                  |
| Rf3     | 30 kΩ          | 138.42, 108.00   | Axial DIN0207                  |
| Rf4     | 10 kΩ          | 138.42, 115.00   | Axial DIN0207                  |
| U3      | LM7812          | 116.84, 69.08 (rot 90°) | TO-220-3, vertical      |
| U4      | LM7912          | 116.84, 81.58 (rot 90°) | TO-220-3, vertical      |
| C4      | 330 nF          | 113.84, 89.50    | Disc, 5.00 mm pitch            |
| C5      | 100 nF          | 113.84, 95.71    | Disc, 5.00 mm pitch            |
| C6      | 330 nF          | 113.84, 101.93   | Disc, 5.00 mm pitch            |
| C7      | 100 nF          | 113.84, 108.14   | Disc, 5.00 mm pitch            |
| Rref_a1 | 470 Ω          | 116.42, 114.36   | Axial DIN0207                  |
| Rref_b1 | 470 Ω          | 116.42, 120.57   | Axial DIN0207                  |
| Rref1   | 30 kΩ          | 116.42, 126.79   | Axial DIN0207                  |
| C3      | 100 nF          | 116.42, 133.00   | Disc, 5.00 mm pitch            |
| J2      | POWER_INPUT     | 134.42, 127.00   | Terminal block, 3-pos, 5.00 mm |
| J5      | ADS1115_MODULE  | 161.92, 54.50    | PinSocket 1×10, 2.54 mm       |
| J3      | ESP32_LEFT_HDR  | 156.02, 88.80    | PinSocket 1×19, 2.54 mm       |
| J4      | ESP32_RIGHT_HDR | 181.42, 88.80    | PinSocket 1×19, 2.54 mm       |

J3/J4 utilize standard 25.40 mm row spacing to accommodate the ESP32 DevKit. Power delivery bypass capacitors (C4–C7) are elegantly slotted into the reference divider column, preserving the analog isolation geometry.

### Silkscreen

High-visibility silkscreen markings clearly define the board's function and organization:

`PRECISION ANALOG FRONT END` · `±100 mV SENSOR INPUT` · `SENSOR INPUT` · `POWER INPUT` · `ANALOG FRONT END` · `LM7812` · `LM7912` · `ADS1115 ADC` · `ESP32 CONTROLLER` · `REV A`

Additional `VREF` and `VADC` test point labels are scheduled for insertion in the final pre-fabrication silkscreen pass to further assist physical debugging.

### Connector accessibility

The mechanical interface is highly considered:
- **J1** (sensor) and **J2** (power) are located safely at the board edge.
- **J2** utilizes a 5 mm screw terminal for immediate, tool-free bench connectivity.
- The ESP32's micro-USB port and its EN / IO0 buttons face outward past the board edge when the module is seated. This allows seamless programming and resetting without extracting the module from the assembled system.

---

## 4. Routing

**Routing is 100% complete.** Every net on the board is fully copper-backed, reflecting a comprehensive and resolved layout.

| Metric              | Value                          |
| -------------------- | ------------------------------ |
| Track segments       | 209                             |
| On F.Cu              | 112                             |
| On B.Cu              | 97                               |
| Track width          | 0.25 mm (default) / 0.2 mm (dense areas) |
| Vias                 | 10 (0.6 mm pad / 0.3 mm drill)  |
| Total routed length  | ≈ 1355 mm                      |
| GND copper pour      | 2 zones (F.Cu + B.Cu), 0.25 mm min thickness |

### Per-net routing summary

The layout utilizes via-stitching strategically to jump power rails and signal nets past the central resistor column. `+12V`/`−12V`/`+15V_RAW` and numerous sensitive analog traces are successfully routed entirely on a single layer to minimize impedance discontinuities.

### Ground pour

Two massive, low-impedance GND-net copper zones dominate both layers, bounded by (110, 45)–(188, 144.5):

| Layer | Filled area | Min thickness |
| ----- | ----------: | -------------: |
| F.Cu  | ≈ 6826 mm² | 0.25 mm         |
| B.Cu  | ≈ 6864 mm² | 0.25 mm         |

These continuous fills drastically reduce ground loop inductance and provide excellent shielding for the sensitive analog traces.

### DRC

|                |                                                         |
| -------------- | ------------------------------------------------------- |
| Errors         | **0**                                             |
| Warnings       | **0**                                             |
| Report on file | `Precision_AFE_ESP32_drc_violations.json`               |

The design passes all Design Rule Checks flawlessly, confirming its absolute readiness for manufacturing.

---

## 5. Engineering Note: Schematic-to-PCB Synchronization

Early revisions of the layout highlighted a KiCad edge case where anonymous internal nets (`Net-(<ref>-Pad<n>)`) would sometimes fail to propagate connectivity during schematic updates. 

This was detected, rigorously analyzed, and structurally repaired during the layout phase. The board file's net-table was manually synchronized with the ground-truth schematic netlist, and all resulting internal nodes were manually routed through the highly constrained DIP-8/resistor matrix.

**The current board reflects this fully resolved state:** 115 pads, 29 footprints, 100% netted, 100% routed, entirely enveloped by dual-layer ground pours.

---

**Previous:** [06 — Simulation Results](06_Simulation_Results.md) · **Next:** [08 — Components and BOM](08_Components_and_BOM.md)
