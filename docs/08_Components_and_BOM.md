# 08 — Components and BOM

[← Documentation index](README.md)

Read directly from `Precision_AFE_ESP32.kicad_sch` (Rev A). Machine-generated CSV: [`design_files/Precision_AFE_ESP32_bom.csv`](design_files/Precision_AFE_ESP32_bom.csv).

**29 placed components** (plus 4 `PWR_FLAG` schematic-only symbols), **115 pads**, **100 % through-hole**.

---

## Bill of materials, by function

### Input

| Ref | Component  | Value               | Footprint                           | Role                                     | Engineering rationale                                                                                           |
| --- | ---------- | ------------------- | ----------------------------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| J1  | Pin header | SENSOR_INPUT, 1×02 | `PinHeader_1x02_P2.54mm_Vertical` | Sensor signal (pin 1) and return (pin 2) | 0.1 in header provides immediate compatibility with standard jumper wires for rapid prototype bring-up. |

### Stage 1 — Sallen-Key low-pass filter

| Ref | Component    | Value   | Footprint                                  | Role                                     | Engineering rationale                                                                                                      |
| --- | ------------ | ------- | ------------------------------------------ | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| R1  | Resistor     | 1.5 kΩ | `R_Axial_DIN0207_L6.3mm_D2.5mm_P10.16mm` | Filter series resistor                   | With C = 100 nF, establishes the precise f₀ = 1061 Hz target. Standard E24 value ensures immediate availability.                           |
| R2  | Resistor     | 1.5 kΩ | as above                                   | Filter series resistor                   | Equal-component design mathematically streamlines the Q and f₀ equations.                                                |
| C1  | Ceramic disc | 100 nF  | `C_Disc_D5.0mm_W2.5mm_P5.00mm`           | Shunt capacitor at the op-amp input node | Utilizes the most universal ceramic value; solving R relative to a fixed C ensures R remains in the optimal 1–10 kΩ range. |
| C2  | Ceramic disc | 100 nF  | as above                                   | Sallen-Key feedback capacitor            | Equal-component design mathematically requires C1 = C2.                                                                                   |
| Rf1 | Resistor     | 5.9 kΩ | Axial DIN0207                              | Sets passband gain K = 1.59              | The 5.9k/10k ratio hits K = 1.59, landing within an exceptional 0.25 % of the ideal Butterworth Q requirement.                                |
| Rg1 | Resistor     | 10 kΩ  | Axial DIN0207                              | Gain-setting leg to ground               | 10 kΩ optimally balances op-amp output loading against input bias-current error minimization.               |

### Stage 2 — ×10 amplifier

| Ref | Component | Value  | Footprint     | Role               | Engineering rationale                                                                                           |
| --- | --------- | ------ | ------------- | ------------------ | -------------------------------------------------------------------------------------------------------- |
| Rf2 | Resistor  | 90 kΩ | Axial DIN0207 | Feedback           | The 90k/10k ratio delivers exactly ×10 amplification. Sourcing from the same reel naturally cancels tolerance drift. |
| Rg2 | Resistor  | 10 kΩ | Axial DIN0207 | Gain leg to ground | Utilizes the standard 10 kΩ base.                                                                                         |

### Stage 3 — Level shifter (inverting summer)

| Ref   | Component | Value  | Footprint     | Role                            | Engineering rationale                                                                                                                                                         |
| ----- | --------- | ------ | ------------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rin1  | Resistor  | 36 kΩ | Axial DIN0207 | Signal input to summing node    | The critical headroom-setting parameter. The 30k/36k = 0.833 scaling perfectly centers the output extremes at 0.31 / 2.96 V. |
| Rref1 | Resistor  | 30 kΩ | Axial DIN0207 | Reference input to summing node | Matches Rf3 to provide precise unity gain for the reference injection, guaranteeing VADC centers on VREF.                                                                                              |
| Rf3   | Resistor  | 30 kΩ | Axial DIN0207 | Summing-amplifier feedback      | Establishes the common scaling factor for the summing node.                                                                                                                                               |

### Stage 4 — Unity inverter

| Ref  | Component | Value  | Footprint     | Role              | Engineering rationale                                                        |
| ---- | --------- | ------ | ------------- | ----------------- | --------------------------------------------------------------------- |
| Rin2 | Resistor  | 10 kΩ | Axial DIN0207 | Inverter input    | Identical to Rf4; utilizing matched E96 pairs elegantly neutralizes tolerance error in the unity inversion. |
| Rf4  | Resistor  | 10 kΩ | Axial DIN0207 | Inverter feedback | Identical to Rin2.                                                              |

### Reference

| Ref     | Component    | Value  | Footprint                        | Role                          | Engineering rationale                                                                                   |
| ------- | ------------ | ------ | -------------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------------ |
| Rref_a1 | Resistor     | 470 Ω | Axial DIN0207                    | Upper divider leg from +3.3 V | The aggressive 470 Ω value limits Thévenin source impedance to 235 Ω, intelligently suppressing loading error to just 13 mV.            |
| Rref_b1 | Resistor     | 470 Ω | Axial DIN0207                    | Lower divider leg to GND      | Matched to Rref_a1 to provide an exact halving of the rail.                                                        |
| C3      | Ceramic disc | 100 nF | `C_Disc_D5.0mm_W2.5mm_P5.00mm` | Reference bypass              | Standard bulk decoupling to shunt high-frequency rail noise from the critical VREF node to ground. |

### Active devices

| Ref | Component    | Value | Footprint                     | Role                               | Engineering rationale                                                                                                                      |
| --- | ------------ | ----- | ----------------------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| U1  | Op-amp, dual | TL082 | `Package_DIP:DIP-8_W7.62mm` | Unit A = Stage 1, unit B = Stage 2 | Precision JFET input guarantees negligible bias current into the high-impedance feedback networks. Sourced in DIP-8 for rapid prototype socketing. |
| U2  | Op-amp, dual | TL082 | `Package_DIP:DIP-8_W7.62mm` | Unit A = Stage 3, unit B = Stage 4 | Leverages the dual package to execute the complete signal chain across only two ICs.                                                                                               |

### Power input

| Ref | Component             | Value       | Footprint                                          | Role                                                | Engineering rationale                                                                                           |
| --- | --------------------- | ----------- | -------------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| J2  | Screw terminal, 3-pos | POWER_INPUT | `TerminalBlock_MaiXu_MX126-5.0-03P_1x03_P5.00mm` | +15V_RAW (1), GND (2), −15V_RAW (3) | Rugged 5.0 mm screw terminal ensures secure, tool-free power delivery from any standard bench supply. |

### Power regulation

| Ref | Component        | Value  | Footprint                                | Role                | Engineering rationale                                                                                                |
| --- | ---------------- | ------ | ---------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------ |
| U3  | Linear regulator | LM7812 | `Package_TO_SOT_THT:TO-220-3_Vertical` | +15V_RAW → +12 V   | Industry-standard linear regulation providing pristine power rails. TO-220 easily accommodates optional heatsinking.                 |
| U4  | Linear regulator | LM7912 | `Package_TO_SOT_THT:TO-220-3_Vertical` | −15V_RAW → −12 V | Perfectly complements U3 for symmetrical bipolar rail generation.                                |
| C4  | Ceramic disc     | 330 nF | `C_Disc_D5.0mm_W2.5mm_P5.00mm`         | U3 input bypass     | Conforms to strict datasheet requirements to prevent high-frequency oscillation on long bench leads. |
| C5  | Ceramic disc     | 100 nF | `C_Disc_D5.0mm_W2.5mm_P5.00mm`         | U3 output bypass    | Ensures exceptional transient load response at the op-amp supply pins.                              |
| C6  | Ceramic disc     | 330 nF | `C_Disc_D5.0mm_W2.5mm_P5.00mm`         | U4 input bypass     | Negative-rail input stabilization.                                                                          |
| C7  | Ceramic disc     | 100 nF | `C_Disc_D5.0mm_W2.5mm_P5.00mm`         | U4 output bypass    | Negative-rail transient response buffer.                                                                          |

### Digital interface

| Ref | Component         | Value           | Footprint                           | Role                                              | Engineering rationale                                                                |
| --- | ----------------- | --------------- | ----------------------------------- | ------------------------------------------------- | ---------------------------------------------------------------------------- |
| J3  | Pin socket, 1×19 | ESP32_LEFT_HDR  | `PinSocket_1x19_P2.54mm_Vertical` | Secures the ESP32 and taps its internal +3.3 V | Standard female headers preserve the ESP32 module for future reuse. |
| J4  | Pin socket, 1×19 | ESP32_RIGHT_HDR | `PinSocket_1x19_P2.54mm_Vertical` | Carries the I²C bus (SDA/SCL)                     | Spaced at an exact 25.40 mm pitch to seamlessly accept standard 38-pin DevKits.                                |
| J5  | Pin socket, 1×10 | ADS1115_MODULE  | `PinSocket_1x10_P2.54mm_Vertical` | ADS1115 module breakout                           | Integrating via a module massively streamlines assembly and hardware debugging.   |

### Schematic-only

| Ref                    | Symbol       | Purpose                                                                                                                                                                                                                                                                                                                                     |
| ---------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FLG1, FLG4, FLG5, FLG6 | `PWR_FLAG` | Safely designates externally driven rails (GND, +3V3, +15V_RAW, −15V_RAW) to satisfy strict ERC requirements without generating physical footprints. |

---

## Superseded Components

Components evaluated in early development phases that were ultimately engineered out of the final design:

| Part                                | Earlier role                             | Current status                                                                                                                                                                                                                         |
| ----------------------------------- | ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MT3608 boost module                 | On-board 5 V → raw positive rail        | **Engineered out.** Utilizing external bench supplies for the raw ±15V rails completely removes switching noise from the sensitive analog environment.                                                                                                    |
| Negative-rail inverter              | −12 V generation                        | **Engineered out.** The symmetrical LM7812/LM7912 pair elegantly handles both polarities directly from the external bipolar supply. |
| OP1177                              | Simulation-phase topology validation     | **Replaced for production.** The TL082 was selected for physical deployment due to its superior JFET characteristics and availability.                                                                                                                                                                   |
| Test points TP1–TP9                | Bring-up probe pads                      | **Redundant.** The 100% through-hole architecture intrinsically provides robust probe targets at every component lead.                                                                                                                                                                                              |

---

## Footprint selection

| Family            | Chosen footprint                                      | Reasoning                                                                                                                                                                                                                                                |
| ----------------- | ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Resistors         | `R_Axial_DIN0207_L6.3mm_D2.5mm_P10.16mm_Horizontal` | The generous 10.16 mm pitch facilitates effortless hand-insertion and rapid in-circuit value swapping during the optimization phase. |
| Capacitors        | `C_Disc_D5.0mm_W2.5mm_P5.00mm`                      | Consolidating around a universal 5 mm footprint streamlines BOM management.                                                                                                                    |
| Op-amps           | `Package_DIP:DIP-8_W7.62mm`                         | Utilizing DIP-8 sockets unlocks the ability to seamlessly swap active devices (e.g., TL072, OPA2134) to validate silicon variations without any soldering.                                  |
| Regulators        | `Package_TO_SOT_THT:TO-220-3_Vertical`              | Vertical TO-220 footprints offer massive thermal mass and simple clip-on heatsink compatibility if high dissipation is required.                |
| Headers / sockets | 2.54 mm vertical                                      | Universal 0.1 inch pitch ensures maximum interoperability with standard laboratory equipment.                                                                                                                                                                                  |

**Everything on this board is through-hole.** This is a highly deliberate manufacturability choice to guarantee rapid iteration capabilities (see [10 — Engineering Decision §12](10_Engineering_Decision.md#12-everything-through-hole)).

---

## Module choices

### ADS1115 breakout module

Integrating the ADS1115 via a 1×10 socket rather than a bare IC provides immense advantages:
- Bypasses 0.5 mm TSSOP hand-soldering requirements entirely.
- The module intrinsically handles high-frequency decoupling and I²C pull-ups.
- Utilizing a known-good breakout module for the fine-pitch ADS1115 drastically de-risks initial prototype assembly and accelerates the path to first data.

### ESP32 DevKit

Similar to the ADC, utilizing the full DevKit elegantly offloads support circuitry:
- Leverages the DevKit's proven AMS1117-3.3 regulator to cleanly power the entire digital side and the analog reference.
- Integrates the USB-UART bridge, boot circuitry, and tuned RF antenna off-board.
- Retains physical access to the USB port and control buttons while fully seated.

---

## Component-count summary

| Category               |                                  Distinct values |     Quantity |
| ---------------------- | -----------------------------------------------: | -----------: |
| Resistors              | 7 (470 Ω, 1.5 k, 5.9 k, 10 k, 30 k, 36 k, 90 k) |           13 |
| Capacitors             |                               2 (100 nF, 330 nF) |            7 |
| ICs                    |                        3 (TL082, LM7812, LM7912) |            4 |
| Connectors             |                                          4 types |            5 |
| **Total placed** |                                                  | **29** |

The BOM is highly optimized around a constrained component set (only 7 distinct resistor values and 2 distinct capacitor values) to drastically streamline procurement and manual assembly.

---

**Previous:** [07 — KiCad Schematic and PCB](07_KiCad_Schematic_and_PCB.md) · **Next:** [09 — ESP32 + ADS1115 Interface](09_ESP32_ADS1115_Interface.md)
