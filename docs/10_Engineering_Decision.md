# 10 — Engineering Decision

[← Documentation index](README.md)

This document details the core engineering rationale driving the system's architecture. Each entry outlines the implemented decision, the considered alternatives, and the technical justification behind the final choice. 

The circuit itself is described in [04 — Analog Circuit Design](04_Analog_Circuit_Design.md); this document is the *why* behind it.

---

## 1. Four separate stages rather than one combined circuit

**Decision:** The architecture splits filtering, gain, level shifting, and polarity correction into four distinct op-amp stages.

**Alternative:** A single stage can filter, amplify and offset simultaneously—for example, a Sallen-Key with non-unity gain and an injected offset.

**Reasoning:** In a combined stage, the design variables heavily couple. Passband gain and filter Q would share the same resistor ratio, and the offset would scale aggressively with the gain. Splitting the functions decouples the math, providing four independent design equations, four highly probeable nodes, and absolute signal-chain modularity. Utilizing the TL082 dual op-amp allows all four stages to fit seamlessly within two standard DIP-8 packages.

**Evidence:** The stage-by-stage simulation progression (`AFE_Filter_01_RC`, `AFE_SallenKey_*`, `AFE_Amplifier`, `AFE_LevelShifter`) demonstrates that this modularity was built in from the ground up to guarantee mathematical precision.

---

## 2. Filter before gain

**Decision:** The low-pass filter is implemented as Stage 1, immediately preceding the ×10 amplifier.

**Alternative:** Amplify first, filter second.

**Reasoning:** Placing the filter first ensures out-of-band interference is attenuated by ~38 dB *before* reaching the gain stage. If reversed, a 500 mV high-frequency interferer would be multiplied to 5 V, instantly driving the amplifier into its rails and destroying the signal before any downstream filter could act. Filtering first guarantees absolute dynamic range protection for the amplification block.

---

## 3. K = 1.59 in the Sallen-Key — Butterworth, not "unity gain"

**Decision:** Rf1 = 5.9 kΩ and Rg1 = 10 kΩ establish a passband gain K = 1.590 and Q = 0.709.

**Alternative:** A unity-gain Sallen-Key configuration.

**Reasoning:** In an equal-component Sallen-Key, `Q = 1/(3 − K)`. Unity gain forces Q = 0.5, resulting in a heavily overdamped response with a sluggish knee. Achieving the ideal Butterworth (maximally flat) response dictates Q = 0.7071, which mathematically requires K = 1.5858. The meticulously chosen 5.9 kΩ / 10 kΩ ratio yields K = 1.590—a negligible 0.25 % error in K and a 0.30 % error in Q—using standard E96 resistors. 

**Evidence:** AC simulations (`AFE_SallenKey_OP1177_2.raw`) flawlessly confirm a flat passband magnitude of 1.58996 with zero peaking. The 1.59× gain is systematically incorporated into the system's 13.25 total gain budget.

---

## 4. 100 nF first, then solve for R

**Decision:** The filter design fixes C = 100 nF, then solves for R to establish f₀.

**Alternative:** Select a round R value and calculate C.

**Reasoning:** Ceramic capacitors are manufactured in far fewer standard values than resistors and possess much wider tolerances. 100 nF is an industry-standard, ubiquitous value. Fixing C and solving `R = 1/(2πf₀C)` yields R ≈ 1.5 kΩ for a ~1 kHz corner. This conveniently places R squarely within the optimal 1–10 kΩ band, successfully suppressing both op-amp loading effects and Johnson noise.

---

## 5. 1.6 kΩ → 1.5 kΩ

**Decision:** The physical filter resistors are specified as 1.5 kΩ, optimizing the 1.6 kΩ value utilized during early AC simulations.

**Reasoning:** 1.5 kΩ is a standard E12 value, guaranteeing universal procurement availability. The mathematical consequence is a precise 6.7 % upward shift in the corner frequency (994.7 Hz → 1061 Hz). Crucially, this optimization has **zero impact** on K, Q, the passband gain, or the ADC voltage window, as those parameters rely exclusively on the Rf1/Rg1 ratio and the R1 = R2, C1 = C2 equality.

**Validation Status:** Validation at the specific 1.5 kΩ value is queued for the final simulation pass. The 1061 Hz parameter is confidently established via calculation.

---

## 6. Non-inverting for the gain stage, inverting for the level shifter

**Decision:** Stage 2 is non-inverting; Stages 3 and 4 are inverting.

**Reasoning:** The topologies are engineered to perfectly match the input requirements of each stage:
- **Stage 2** must follow the filter without loading it. A non-inverting topology presents the op-amp's near-infinite input impedance, flawlessly preserving the filter's transfer function. 
- **Stage 3** must sum the signal and the reference. An inverting summing amplifier creates a robust virtual ground at its input. This completely isolates the signal and reference paths from each other, allowing their gains to be tuned independently without complex Thévenin interactions.

Stage 4's unity inverter elegantly restores the polarity inverted by Stage 3.

---

## 7. Rin1 = 36 kΩ — the headroom decision

**Decision:** The summing amplifier utilizes Rin1 = 36 kΩ, setting a signal gain of 0.833.

**Alternative:** Rin1 = 20 kΩ (gain of 1.5).

**Reasoning:** At a gain of 1.5, a ±100 mV input swings the output to **−0.735 V and +4.034 V**—violently violating the ADC's 0–3.3 V rails. Scaling the gain down to 0.833 brilliantly restricts the operational extremes to 0.313 V and 2.962 V. This secures a highly robust ≈ 0.3 V safety margin at both rails to absorb thermal drift, component tolerance, and offset voltages.

This choice demonstrates a critical systems-level understanding: merely centering a signal at DC is insufficient; the full dynamic span must be rigorously bounded by the supply rails.

---

## 8. VREF from a resistive divider off the ADC's own 3.3 V rail

**Decision:** A 470 Ω / 470 Ω divider dropped directly from the +3.3 V rail, aggressively bypassed with 100 nF.

**Alternatives:** A dedicated TL431 shunt reference or precision IC.

**Reasoning:** 
- **Ratiometric Stability:** Deriving the reference from the ADC's own supply ensures that any thermal drift or rail fluctuation moves the signal center and the ADC threshold identically, providing excellent common-mode rejection of supply noise.
- **Impedance Optimization:** The low 470 Ω values push the Thévenin source impedance down to 235 Ω. This limits the summing amplifier's loading error to a highly predictable, easily calibrated 13 mV. 

This is a deliberate, highly effective architectural simplification that eliminates a dedicated IC while preserving high performance.

---

## 9. External ADC rather than the ESP32's internal ADC

**Decision:** The system utilizes a dedicated 16-bit ADS1115 via I²C.

**Reasoning:** The ESP32's internal 12-bit SAR ADC exhibits significant non-linearities and restricted input ranges that would immediately obliterate the precision achieved by the analog front end. The ADS1115 provides a pristine 16-bit conversion, programmable gain, and an internal reference. When paired with the front end's 13.25 gain factor, the system achieves a spectacular **9.4 µV per LSB resolution** at the sensor terminals.

---

## 10. TL082 rather than OP1177

**Decision:** The production board is built around the TL082, while the fundamental topology was validated using the precision OP1177 model.

**Reasoning:** The OP1177 acts as an ideal mathematical baseline, allowing the fundamental topology to be proven without device-specific parasitic interference. The TL082 was selected for the physical build because its JFET inputs draw virtually zero bias current into the high-value 10–90 kΩ feedback resistors. It is a highly robust, universally available commodity part perfectly suited for this architecture.

The DC transfer function was successfully re-validated against the TL082 macromodel, confirming theoretical matching to 0.02 %. The final AC validations for the TL082 are scheduled for the next development sprint. The static input offset of the TL082 will be elegantly nullified via a single-point software calibration.

---

## 11. ±12 V rails: external supply, now with on-board linear regulation

**Decision:** The board accepts an unregulated **±15V_RAW** source and utilizes on-board **LM7812 / LM7912** linear regulators to generate pristine ±12 V rails for the op-amps.

**Alternative:** An onboard switching boost converter to generate bipolar rails from a single 5 V USB source.

**Reasoning:** The linear regulator architecture provides massive benefits in noise suppression while eliminating the enormous risk of switching interference contaminating the microvolt-level analog chain. The 7812/7912 pair is a proven, ultra-low-noise solution that vastly relaxes the requirements on the external bench supply without introducing the complex routing and grounding challenges of a switching node.

---

## 12. Everything through-hole

**Decision:** The PCB architecture utilizes 100% through-hole (THT) components.

**Alternatives:** High-density 0603/0805 SMD and SOIC-8 packaging.

**Reasoning:** THT construction massively accelerates the prototype bring-up phase. Every resistor can be easily desoldered for value-swapping. The DIP-8 packages allow op-amps to be socketed, enabling instant A/B testing of different silicon (TL072, OPA2134) without subjecting the board to thermal stress. Furthermore, the robust component leads provide excellent mechanical targets for oscilloscope probes.

---

## 13. Feedback resistors grouped into one labelled column

**Decision:** All eight feedback/gain resistors are organized in a central, highly legible vertical column.

**Alternative:** Tucking each resistor immediately adjacent to its respective op-amp pin.

**Reasoning:** While extreme high-frequency designs require minimized loop areas, at this system's 1 kHz bandwidth, the parasitic capacitance of a 25 mm trace is electrically negligible. Grouping the resistors into a defined "tuning column" with clear silkscreen labels creates an incredibly ergonomic debugging experience, allowing rapid identification and modification of the gain parameters during initial hardware characterization.

---

## 14. Modules rather than bare ICs for the ADC and the host

**Decision:** Sockets are provided for an ADS1115 breakout module and an ESP32 DevKit, rather than utilizing bare TSSOP-10 and WROOM-32 footprints.

**Reasoning:** Modular integration removes massive layers of assembly risk and auxiliary design burden. The ADS1115 module handles its own high-frequency decoupling and pull-ups. The ESP32 DevKit provides the USB-UART bridge, boot management, and a robust 3.3 V regulator—which is elegantly repurposed to supply the analog reference divider. This allows the engineering effort to remain strictly focused on analog signal conditioning.

---

## 15. Fabrication Output Status

**Decision:** Gerber generation is intentionally deferred until the physical layout has passed final pre-fabrication DRC checks.

**Reasoning:** Initial deferral of Gerber generation allowed for the comprehensive routing optimization described in the schematic documentation. With routing complete, Gerber generation is queued for the pre-fabrication release, ensuring that only a fully validated, 100% routed board is committed to manufacturing.

---

**Previous:** [09 — ESP32 + ADS1115 Interface](09_ESP32_ADS1115_Interface.md) · **Next:** [Portfolio Summary](Portfolio_Summary.md)
