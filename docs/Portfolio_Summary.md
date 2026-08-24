# Portfolio Summary

[← Documentation index](README.md)

---

## Precision Mixed-Signal Data Acquisition Front End (AFE)

**Analog signal conditioning for a ±100 mV sensor into a 16-bit ADC · KiCad · LTspice · ESP32**

---

## Elevator pitch

A four-stage precision analog front end that takes a ±100 mV bipolar sensor signal and conditions it — filtering, amplification, and level shifting — into the 0–3.3 V unipolar window of a 16-bit ADS1115 ADC read by an ESP32 over I²C. Designed from requirements, verified stage-by-stage in LTspice, and laid out as a 78 × 100 mm two-layer PCB in KiCad. The current development stage establishes the circuit architecture, simulation behavior, and PCB implementation, providing a solid foundation for physical hardware validation.

---

## The problem

A sensor putting out ±100 mV cannot be connected directly to a 3.3 V ADC and measured well. Three things go wrong at once, and each requires its own circuit-level solution:

- The signal uses **2.4 % of the converter's span** — a 16-bit ADC delivers about 5 useful bits.
- Half of every cycle is **below the converter's ground rail** and is simply lost.
- Anything above the sample rate **folds back into the measurement band** and is indistinguishable from real signal.

---

## The solution

I designed a four-stage signal chain built around two dual TL082 op-amps to solve these specific issues:

| Stage | Function | Implementation | Gain |
|---|---|---|---|
| 1 | Band-limit | Sallen-Key Butterworth low-pass, f₀ = 1.06 kHz, Q = 0.709 | ×1.59 |
| 2 | Amplify | Non-inverting amplifier | ×10 |
| 3 | Level shift | Inverting summing amplifier with a 1.65 V reference | ×(−0.833) |
| 4 | Correct polarity | Unity-gain inverter | ×(−1) |

**Result:** ±100 mV in → 0.31 V … 2.96 V out, a gain of 13.25 V/V. The signal sits securely inside the ADC's rails with ~0.3 V of margin at each end, successfully resolving **9.4 µV at the sensor** per ADC code.

---

## Technical implementation

**Analog design.** I synthesized the filter to a specified Q rather than a specified gain. In the equal-component Sallen-Key topology, passband gain and pole Q are tied together through `Q = 1/(3 − K)`. As a result, the 1.59× gain is the intentional *consequence* of targeting Butterworth flatness, and I carried this factor through the chain's gain budget. Level shifting is achieved via an inverting summing junction, where a virtual ground keeps the signal and reference paths independent.

**Quantitative design decisions.** I adjusted the summing amplifier's input resistor from 20 kΩ to 36 kΩ after evaluating the transfer function at its extremes rather than just at zero. The original value centred correctly on 1.65 V but reached −0.74 V and +4.03 V at ±100 mV — outside the ADC's rails at both ends. Additionally, I predicted the reference divider's 13 mV loading error by hand (55 µA through a 235 Ω source) before confirming it at 1.6372 V via simulation.

**Simulation.** I performed stage-by-stage LTspice verification with an OP1177 model, including an **ideal-VCVS control case** run alongside the real op-amp. This revealed that the filter's stopband floors out at ≈ −38 dB above 10 kHz rather than continuing at 40 dB/decade, a behavior the equations alone do not show. I then executed a device-substitution phase: integrating a third-party TL082 PSpice macromodel (utilizing Gmin and source stepping to converge) and re-running the full chain. The final DC transfer matches the closed-form result to **0.015 %**.

**PCB.** The KiCad 10 schematic capture and two-layer through-hole layout translate the validated schematic into a compact hardware implementation. I physically separated the analog and digital sections to maintain a clear signal flow, kept the ESP32's USB and boot buttons accessible, and grouped all gain-setting resistors into one silkscreen-labelled column to streamline bring-up. The layout is DRC clean, and ERC is clean apart from 32 expected header-breakout warnings.

**Documentation.** Fourteen engineering documents, including a verification matrix that separates calculated from simulated values, and a self-audit that cross-checks the schematic against the PCB layout and the LTspice files.

---

## Engineering skills demonstrated

**Analog circuit design**
Active filter synthesis (Sallen-Key, Butterworth Q targeting) · op-amp gain-stage design · inverting summing amplifiers · bipolar-to-unipolar level shifting · resistive reference design with loading analysis · supply-rail headroom budgeting · error-budget and tolerance stack-up analysis

**Circuit simulation**
LTspice AC / transient / DC-sweep analysis · third-party SPICE macromodel integration · convergence troubleshooting (Gmin stepping, source stepping) · ideal-vs-real control-case methodology · netlist-level inspection and raw-file data extraction · model-substitution verification

**PCB design**
KiCad 10 schematic capture · footprint selection · two-layer through-hole layout · functional block placement · analog/digital section separation · design-rule setup · DRC and ERC

**Mixed-signal and embedded**
External ADC selection and interfacing · I²C bus design · ADS1115 address strapping and PGA range selection · ESP32 dev-board integration · resolution and range budgeting from sensor to ADC code

**Engineering process**
Requirement-driven decomposition into independently testable stages · quantitative trade-off analysis with alternatives evaluated · iterative revision with superseded design points preserved for traceability · rigorous separation of design intent from simulation result · self-auditing of design-file consistency

---

## What this project demonstrates

The core engineering value of this project lies in the deliberate decisions underpinning the architecture: *why* filter before gain; *why* the filter's gain is 1.59 and not 1; *why* the level-shifter's input resistor is 36 kΩ and not 20 kΩ; *why* a 470 Ω divider instead of a 10 kΩ one; *why* verify with a precision op-amp and then re-verify with the target device.

Each of those decisions has a calculated value attached, an alternative that was considered and rejected, and simulation verification. The result is a complete, purpose-built analog front end that is thoroughly engineered, well-documented, and ready for physical fabrication and testing.
