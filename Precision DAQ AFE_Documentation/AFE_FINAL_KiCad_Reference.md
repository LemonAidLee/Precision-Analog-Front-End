# Precision Analog Front-End (AFE) — FINAL AUTHORITATIVE REFERENCE
## Consolidated Source of Truth for KiCad Schematic Capture and PCB Design

**Document status:** This is the authoritative consolidation document for the PCB design phase. It supersedes all prior individual documents (`AFE_Master_Engineering_Documentation.md`, `01_Problem_Purpose_and_Value.md`, `02_How_the_Project_Works.md`, `03_Materials_and_Tools.md`) wherever they conflict with this document. Nothing in the underlying engineering has been changed, redesigned, or re-optimized while writing this document — it is a traceability and consolidation exercise only.

**Status legend used throughout this document:**

| Tag | Meaning |
|---|---|
| **VERIFIED** | A real LTspice result (`.op`, `.ac`, or `.tran`) was actually run and its output value directly recorded or extracted from the `.raw`/`.log` file. |
| **CALCULATED** | A value derived by hand/algebra from VERIFIED or ASSUMED values — no fresh simulation was run for this specific number. |
| **ASSUMED** | A designer choice (a target, a requirement, a representative input) that was never measured — e.g. the ±100 mV sensor signal. |
| **NOT VERIFIED** | Nothing has been simulated or measured for this item yet. Do not treat it as safe to build against. |
| **SUPERSEDED** | An earlier value/decision that has been explicitly replaced by a later one. Kept here for traceability — **do not use it.** |

---

## 1. Executive Summary

The AFE is a 4-stage analog signal-conditioning chain (filter → amplifier → level shifter → ADC) that has been fully designed and verified in LTspice, including a full-chain system-level integration test. That integration test caught a real design error — the original level-shifter gain sent the signal 0.7 V outside the ADC's 0–3.3 V window — which was root-caused and fixed by changing exactly one resistor (`Rin1`: 20 kΩ → 36 kΩ). The corrected design was re-simulated end-to-end and is LTspice-VERIFIED to stay within 0.325 V–2.974 V at the ADC input, with margin to spare.

Since that verification, two further changes have occurred that are **not yet reflected as fully re-verified LTspice runs**, and this document exists specifically to make that distinction impossible to miss during KiCad capture:

1. **R1/R2 sourcing substitution (1.6 kΩ → 1.5 kΩ):** CALCULATED safe, but the dedicated 1.5 kΩ LTspice schematics were prepared and **never successfully run** (headless batch simulation did not complete in the working environment — no `.log`/`.raw` output exists for `AFE_SallenKey_R1p5k_test.asc` or `AFE_SallenKey_R1p5k_DCstep_test.asc`). The predicted new cutoff (≈1063.9 Hz) is an extrapolation, not a fresh simulation result.
2. **Op-amp substitution (OP1177 → TL082) for the physical build:** This is a sourcing decision made for cost/availability reasons. **TL082 has never appeared in any LTspice run in this project.** Every simulated result in this document — every gain, every dB figure, every ADC voltage — was generated with the OP1177 SPICE model. TL082 behavior must be re-verified before fabrication (Section 7.5).

The current PCB-bound design also carries two open, NOT VERIFIED items that must be resolved before schematic finalization: the 1.65 V reference sub-circuit (never simulated as a real circuit — only ever modeled as an ideal DC source) and the negative-rail (−12 V) generation method (the MT3608 boost converter previously appearing in the BOM is **not** a negative-voltage solution and that earlier implication is an error — see Section 20).

---

## 2. Project Objective

This is a precision analog front-end for sensors producing small bipolar analog signals, ASSUMED at up to **±100 mV** (a representative design target — no physical sensor has been characterized). The objective is to condition that signal through a chain of analog stages so it becomes a usable, ADC-compatible voltage for a microcontroller.

At this PCB stage, the primary engineering focus is the internal signal-conditioning chain itself:

```
Sensor-level small analog signal (±100 mV, ASSUMED)
        ↓
Filtering            — 2nd-order Sallen-Key low-pass, ~1 kHz cutoff
        ↓
Amplification        — 10× non-inverting precision gain
        ↓
Level shifting / DC offset — bipolar → 0–3.3 V, 1.65 V reference
        ↓
ADC-compatible voltage — 0.325 V – 2.974 V into ADS1115
        ↓
ESP32 / digital system — I²C read-out
```

The sensor itself, the battery, the enclosure, and any application-specific mechanical system are explicitly **not** the focus of this PCB. The engineering story that matters for this board is how each hardware stage transforms the signal.

---

## 3. Complete Signal Chain (current, final)

| Stage | Function | Topology | Status |
|---|---|---|---|
| Sensor input | ±100 mV bipolar source | N/A — external | ASSUMED (no physical sensor characterized) |
| Stage 5 — Filter | 2nd-order low-pass, ~1 kHz | Sallen-Key, single op-amp | VERIFIED (OP1177, R1=R2=1.6 kΩ) / CALCULATED (R1=R2=1.5 kΩ) |
| Stage 6 — Amplifier | 10× gain | Non-inverting, single op-amp | VERIFIED (OP1177) |
| Stage 7 — Level shifter | Bipolar → 0.325–2.974 V | 2× cascaded inverting op-amps (summing + unity inverter) | VERIFIED (OP1177, Rin1 = 36 kΩ) |
| Stage 8 — Integration | Full-chain check | — | VERIFIED (Rin1 = 36 kΩ full-chain re-sim) |
| ADC | Digitizes 0–3.3 V | ADS1115, I²C, 16-bit | NOT VERIFIED (design intent only, no hardware test) |
| MCU | Reads ADC, sends to PC | ESP32 dev board (exact variant unconfirmed) | NOT VERIFIED |

---

## 4. Stage 1–8 Development History

### Stage 1 — Filter Requirements

**Purpose:** Define what the low-pass filter needs to achieve before any circuit is drawn.

**Target (ASSUMED design target, not a hard spec):**
- 2nd-order low-pass filter
- ≈1 kHz cutoff
- Butterworth-like response (maximally flat passband, no peaking)
- ≈−40 dB/decade roll-off above cutoff

**Rationale:** The AFE's sensor signal may contain unwanted high-frequency noise; the filter removes it before amplification so the amplifier doesn't also gain up the noise.

**Result:** No circuit yet — this stage only set the target that Stages 2–6 are validated against. No pass/fail applicable.

---

### Stage 2 — Filter Mathematics

**Purpose:** Hand-calculate cutoff frequency, passband gain, and Q for the chosen component values, before simulating.

**Components (design choice):** R1 = R2 = 1.6 kΩ, C1 = C2 = 100 nF, Rf = 5.9 kΩ, Rg = 10 kΩ.

**Cutoff:**
$$f_c = \frac{1}{2\pi RC} = \frac{1}{2\pi(1.6\,\text{k}\Omega)(100\,\text{nF})} \approx \boxed{994.7\,\text{Hz}}$$
CALCULATED. Close enough to the 1 kHz target to accept.

**Passband gain:**
$$K = 1+\frac{R_f}{R_g} = 1+\frac{5.9\,\text{k}\Omega}{10\,\text{k}\Omega} = 1.59 \;(\approx 4.03\,\text{dB})$$
CALCULATED.

**Q factor:**
$$Q=\frac{1}{3-K}=\frac{1}{3-1.59}\approx\boxed{0.709}$$
CALCULATED. Compared against Butterworth $Q\approx0.707$ — very close.

**Result:** No hardware/simulation yet — this stage is pure math, feeding Stage 3–4. All figures here are CALCULATED, later confirmed by simulation in Stage 5.

---

### Stage 3 — Component Selection (Filter)

| Component | Value | Purpose |
|---|---:|---|
| R1 | 1.6 kΩ | Filter frequency-setting resistor |
| R2 | 1.6 kΩ | Filter frequency-setting resistor |
| C1 | 100 nF | Filter capacitor to ground |
| C2 | 100 nF | Filter feedback capacitor |
| Rf | 5.9 kΩ | Sets op-amp passband gain (K) |
| Rg | 10 kΩ | Sets op-amp passband gain (K) |
| Op-amp | OP1177 | Precision amplifier (SPICE model) |
| Supply | +12 V / −12 V | Op-amp supply |

1% resistors were selected so the calculated cutoff frequency and passband gain hold up in practice. **This is the original R1=R2=1.6 kΩ selection — later superseded to 1.5 kΩ for sourcing reasons (Section 6, Stage 21 addendum). Do not build with 1.6 kΩ.**

---

### Stage 4 — Sallen-Key Circuit Design (Topology)

**Purpose:** Translate the Stage 2/3 math into an actual Sallen-Key active-filter topology using a single op-amp.

**Structure:**
```
Vin → R1 → Node A → R2 → Node B → OP1177 non-inverting input
                                      ↓ C1
                                     GND
Node A → C2 → Vout

Op-amp gain network:
Vout → Rf → inverting input → Rg → GND

Power: +12 V / −12 V
```

R1/R2 and C1/C2 set the filter frequency; Rf and Rg set the gain (K) and, through K, the Q factor; the OP1177 actively buffers the output and prevents loading.

**Figure — Stage 4 topology.** The original documentation references an embedded screenshot (`Pasted image 20260820120740.png`) for the Sallen-Key layout. **That image file was not found anywhere in the current project directory** — it is a broken/missing asset in the source Obsidian vault. Do not treat any redrawn version of it as verified; if a topology diagram is needed for KiCad reference, redraw it directly from the netlist structure above and from `AFE_SallenKey_OP1177_2.asc`.

**Result:** Topology only — no simulation yet. NOT VERIFIED as a standalone claim; verification happens in Stage 5.

---

### Stage 5 — Low-Pass Filter Simulation & Validation

**Purpose:** Actually simulate the Stage 4 topology in LTspice and confirm it hits the Stage 1 target.

**5.1 — AC analysis** (`.ac dec 100 10Hz 100kHz`)

| Metric | Theoretical | VERIFIED (LTspice) |
|---|---:|---:|
| Passband gain | 4.03 dB | ≈4.03 dB |
| −3 dB frequency | ≈998 Hz | ≈997.38 Hz |

Error vs. 1 kHz target: $\frac{|1000-997.38|}{1000}\times100\%\approx0.26\%$. **PASS.**

*Figure 1 — `AFE_SallenKey_OP1177_BodePlotAnalysis.png`. AC sweep of the real-OP1177 filter, showing the ≈997 Hz −3 dB point. Proves the built circuit matches the Stage 2 calculation to within 0.26%. Condition: `.ac dec 100 10Hz 100kHz`, R1=R2=1.6 kΩ, OP1177.*

**5.2 — Ideal E-source verification** (isolates filter topology from real op-amp non-idealities)

| Frequency | VERIFIED attenuation |
|---:|---:|
| ≈10.028 kHz | −36.1126 dB |
| 100 kHz | −76.0604 dB |

Roll-off ≈ −39.95 dB/decade (CALCULATED from the two VERIFIED points above) vs. −40 dB/decade theoretical. **PASS** — confirms genuine 2nd-order behavior.

*Figure 2 — `BodePlot.png`. Ideal E-source AC sweep. Proves the filter topology itself (independent of op-amp non-idealities) is a true 2nd-order low-pass. Condition: ideal voltage-controlled source replacing OP1177.*

**5.3 — Real OP1177 verification**

Passband ≈4.03 dB, −3 dB freq ≈997 Hz — PASS for the ≈1 kHz operating region. At 10 kHz: ≈−36.21 dB; at 100 kHz: ≈−38.15 dB — this flattening at high frequency is the **real op-amp's finite bandwidth showing through**, not the filter's true roll-off. Do not misread this as the filter degrading to ≈−1.94 dB/decade.

*Figure 3 — `AFE_SallenKey_OP1177_BodePlotAnalysis.png` (same figure as Fig. 1, real-OP1177 sweep). Proves realistic high-frequency deviation from the ideal 40 dB/decade curve due to OP1177's own bandwidth limit.*

**5.4 — Transient analysis** (three tests, all VERIFIED, all PASS)

| Test | Input | Expected | VERIFIED result | Result |
|---|---|---|---|---|
| 100 Hz (`SINE(0 1 100)`, `.tran 0 50m 0 10u`) | ±1 V | Full passband gain | Output ≈±1.59 V (gain ≈1.59×) | PASS |
| 1 kHz (`SINE(0 1 1000)`, `.tran 0 10m 0 1u`) | ±1 V | ≈1.125 V peak (CALCULATED: $1.59/\sqrt2$) | ≈1.12–1.15 V peak | PASS |
| 10 kHz (`SINE(0 1 10000)`, `.tran 0 2m 0 100n`) | ±1 V | Strong attenuation | Large startup transient settling to strongly attenuated steady state | PASS |

*Figure 4 — `Transient_100Hz.png`. Full passband gain at 100 Hz (well below cutoff).*
*Figure 5 — `Transient_1kHz.png`. ≈3 dB attenuation at the cutoff frequency.*
*Figure 6 — `Transient_10kHz.png`. Strong steady-state attenuation at 10× cutoff, after an initial capacitor-charging transient (not to be confused with the steady-state filter response).*

**5.5 — Cross-check against actual full-chain measured data (Stage 8A)**

Using the real ±100 mV sensor signal through the real chain (VERIFIED, see Section 9.1):
- 100 Hz (passband): $V_{filter}\approx158.99\,\text{mV}$
- 1 kHz (cutoff): $V_{filter}\approx112.00\,\text{mV}$ → $20\log_{10}(112/158.99)\approx-3.04\,\text{dB}$ (CALCULATED), consistent with the theoretical −3.01 dB Butterworth definition.
- 10 kHz: $V_{filter}\approx1.536\,\text{mV}$ → −36.27 dB rel. sensor, −40.30 dB rel. passband (CALCULATED); the two figures reconcile ($-40.30+4.03\approx-36.27$), confirming strong high-frequency rejection.

**Stage 5 conclusion: COMPLETE, PASS with R1=R2=1.6 kΩ.** Lesson learned: use an ideal E-source to separate "does the topology work" from "does the real op-amp behave ideally" — this made the Stage 5.3 high-frequency deviation immediately explainable instead of alarming. **R1/R2 subsequently changed to 1.5 kΩ for sourcing reasons — see Section 6 below; this does not reopen Stage 5's PASS verdict, but the exact −3 dB frequency number above applies only to the 1.6 kΩ build.**

---

### Stage 6 — Precision Amplifier Design and Simulation

**Purpose:** Take the filtered signal and amplify it ×10 while preserving waveform shape, using a non-inverting topology (chosen for its high input impedance, so it doesn't load the filter's output).

**Design requirement:** ASSUMED $V_{in}=\pm100\,\text{mV}$ (representative sensor level, never physically measured) → target $V_{out}=\pm1\,\text{V}$ → required gain $A_v=10$.

**Gain equation and components:**
$$A_v=1+\frac{R_F}{R_G}=1+\frac{90\,\text{k}\Omega}{10\,\text{k}\Omega}=10$$

| Component | Value | Purpose |
|---|---:|---|
| OP1177 | — | Precision op-amp |
| Rf | 90 kΩ | Feedback resistor, sets gain |
| Rg | 10 kΩ | Ground-referenced resistor, sets gain |
| Supply | ±12 V | — |

**Tests (all VERIFIED against the OP1177 SPICE model, all PASS):**

| # | Test | Input | Expected | VERIFIED result | Result |
|---|---|---|---:|---:|---|
| 1 | +DC gain | +0.1 V | +1 V | +0.999638 V ($A_v=9.99638$, error 0.036%) | PASS |
| 2 | −DC gain | −0.1 V | −1 V | −1.00001 V ($A_v\approx10.0001$) | PASS |
| 3 | 100 Hz transient | `SINE(0 0.1 100)` | ≈1 V peak | clean, no clipping | PASS |
| 4 | 1 kHz transient | `SINE(0 0.1 1000)` | ≈1 V peak | clean, no distortion | PASS |
| 5 | AC gain & bandwidth | `.ac dec 100 1 10Meg` | 20 dB passband | $f_{-3dB}=149.43921\,\text{kHz}$; $GBW\approx1.49\,\text{MHz}$ (CALCULATED estimate from the closed-loop sim, **not** a datasheet spec) | PASS |
| 6 | Max input / clipping stress | ±1 V (deliberately overdriven) | ±10 V ideal | ≈±10 V, no obvious clipping, ≈2 V headroom to ±12 V rails | PASS (SPICE-model result only — not a physical hardware claim) |
| 7A | Slew-rate / 10 kHz | `SINE(0 0.1 10000)` | ≈±1 V | Clean, no visible slew-rate distortion. Required SR (CALCULATED) ≈0.0628 V/µs | PASS |

*Figure 7 — `Amplifier_100Hz.png`. Clean ≈10× gain at 100 Hz.*
*Figure 8 — `Amplifier_1kHz.png`. Clean ≈10× gain at 1 kHz, confirming the amplifier itself doesn't distort before the filter interaction is considered.*
*Figure 9 — `Amplifie_AcGain&Bandwidth.png`. AC sweep showing ≈20 dB passband gain and 149.4 kHz bandwidth — ≈149× the filter's own operating region, so the amplifier is never the bandwidth bottleneck.*

**Cross-check against actual full-chain measured data (Stage 8B):** $A_v = V_{amp}/V_{filter} = 1.58946/0.158993 \approx 9.997$ (target 10, error ≈0.03%). CALCULATED from VERIFIED values.

**Stage 6 conclusion: COMPLETE, PASS. Not changed in the final redesign.** Lesson learned (important, later relevant to Stage 8): the amplifier was tested against its **own** ±100 mV assumption directly, bypassing the filter — correct for isolating the amplifier, but it meant the amplifier was never tested against what the filter would *actually* hand it in the full chain. **Physical hardware validation has not yet been performed — Stage 6 is simulation-validated only, and every number above used the OP1177 model, not TL082.**

---

### Stage 7 — Level Shifter Design and Simulation

**Purpose:** Convert the bipolar amplified signal into a voltage the ADC can accept (0–3.3 V), centered on a 1.65 V reference.

**7.1 — Original topology (AS FIRST SIMULATED — SUPERSEDED, do not build this way)**

Two cascaded inverting OP1177 stages:
- **Stage A (U3, summing inverter):** sums the amplifier signal and a 1.65 V DC reference at a shared virtual-ground node.
- **Stage B (U4, unity inverter):** un-inverts Stage A's output back to correct polarity.

| Component | Original value | Role |
|---|---:|---|
| Rin1 | **20 kΩ** ← SUPERSEDED | Signal input resistor (summing node) |
| Rref1 | 30 kΩ | Reference input resistor |
| Rf3 | 30 kΩ | Feedback, U3 |
| VREF1 | DC 1.65 V (ideal source) | Reference — never itself simulated as a real circuit |
| Rin2 | 10 kΩ | Input resistor, U4 |
| Rf4 | 10 kΩ | Feedback, U4 |
| Op-amps | OP1177 ×2 | — |
| Supply | ±12 V | — |

Original transfer function (CALCULATED from the resistor values, matched by LTspice at the time):
$$V_{ADC}=1.5\,V_{AMP}+1.65\,\text{V}$$

**Standalone test:** a 10 kHz signal applied directly to the level shifter's input (bypassing filter/amplifier) VERIFIED $V_{ADC}=1.5V_{in}+1.65$ correctly in isolation. **PASS in isolation — but this is exactly the critical caveat that caused the Stage 8 failure below: it was never tested against the *actual* amplitude Stage 6 delivers.**

*Figure 10 — `AFE_LevelShifter_Circuit.png`. Original Stage 7 schematic (Rin1 = 20 kΩ). Historical reference only — SUPERSEDED, do not use for the physical build.*

**7.2 — FINAL redesigned level shifter — VERIFIED**

Following the Stage 8 system failure (Section 4, Stage 8/10) and root-cause analysis, **only `Rin1` was changed**: 20 kΩ → **36 kΩ**. All other level-shifter components are unchanged.

$$\text{Level-shifter signal gain}=\frac{R_{f3}}{R_{in1}}=\frac{30\,\text{k}\Omega}{36\,\text{k}\Omega}\approx0.8333\times$$

**Final transfer function (VERIFIED via full-chain LTspice re-simulation, netlist `AFE_IntegrationComplete.net`, dated 2026-08-21 15:05:47):**
$$\boxed{V_{ADC}=0.8333\,V_{AMP}+1.65\,\text{V}}$$

This is the current, final, authoritative Stage 7 transfer function. See Section 4/Stage 8 and Section 6 (Level-Shifter Design) below for the full verification data.

**Stage 7 conclusion:** Original topology PASS in isolation but led to a system-level failure when chained (Stage 8). Corrected design (Rin1=36 kΩ) is VERIFIED end-to-end. **Physical hardware validation has not been performed — every number here used OP1177, not TL082.**

---

### Stage 8 — Full System Integration

**8A — Filter + amplifier chain, AC/transient sweep (level shifter not yet included).** VERIFIED, all PASS:

| Frequency | $V_{sensor}$ (mV pk) | $V_{filter}$ (mV pk) | $V_{amp}$ (mV pk) | Filter gain | Total gain | Atten. rel. passband | Result |
|---:|---:|---:|---:|---:|---:|---:|---|
| 100 Hz | 100 | 158.99 | 1589 | 1.590 | 15.89 | 0.00 dB (ref) | PASS |
| 500 Hz | 100 | 154.33 | 1543 | 1.543 | 15.43 | −0.26 dB | PASS |
| 1 kHz | 100 | 112.00 | 1120 | 1.120 | 11.20 | −3.04 dB | PASS |
| 2 kHz | 100 | 38.16 | 381 | 0.382 | 3.81 | −12.40 dB | PASS |
| 10 kHz | 100 | 1.536 | 15.14 | 0.0154 | 0.151 | −40.30 dB | PASS |

Roll-off between 1 kHz and 10 kHz (CALCULATED): ≈−37.3 dB/decade — consistent with the expected −40 dB/decade 2nd-order behavior (small deviation because 1 kHz sits almost exactly at the corner, and because the real OP1177 has finite bandwidth, per Stage 5.3).

*Figure 11 — `Integration_1kHz.png`. Stage 8A at 1 kHz: $V_{sensor}$ (green, ±0.1 V), $V_{filter}$ (blue, ≈±0.11 V), $V_{amp}$ (red, ≈±1.12 V).*
*Figure 12 — `Integration_10kHz.png`. Stage 8A at 10 kHz: startup transient settling to a strongly attenuated steady state ($V_{amp}\approx\pm15\,\text{mV}$, red).*

**8B — Full-chain DC range test, ORIGINAL design (Rin1 = 20 kΩ) — SUPERSEDED / FAILED**

| Sensor input | $V_{filter}$ | $V_{amp}$ | $V_{ADC}$ | Within 0–3.3 V? |
|---|---:|---:|---:|:---:|
| 0 V | −2.24422 µV | −0.000206307 V | **1.64957 V** | Yes |
| +100 mV | +0.158993 V | +1.58946 V | **+4.03384 V** | **NO — 0.734 V over the top rail** |
| −100 mV | −0.158998 V | −1.58988 V | **−0.734703 V** | **NO — 0.735 V below ground** |

**Verdict: FAIL.** All values VERIFIED (direct LTspice op-point results). **These are the SUPERSEDED, original-design ADC figures — do not use −0.735 V / +4.034 V as the final ADC range for anything.** Preserved here strictly for traceability of the design-error story.

*Figure 13 — `AFE_CompleteIntegratedCircuit.png`. Full-chain schematic used for the Stage 8B sweep, showing the original (failing) Rin1 = 20 kΩ.*

**Root cause (CALCULATED):** individual stage gains were never multiplied end-to-end before component selection.
$$A_{total}=A_{filter}\times A_{amplifier}\times A_{level\,shifter}=1.59\times10\times1.5=23.85$$
$$\pm100\,\text{mV}\times23.85\approx\pm2.385\,\text{V}\;\Rightarrow\;V_{ADC}=1.65\,\text{V}\pm2.385\,\text{V}=-0.735\,\text{V to }+4.035\,\text{V}$$
This matches the VERIFIED measurement almost exactly. **Every stage passed its own isolated test; no single test ever checked the actual chained amplitude against the ADC's fixed window.** This was a system-level gain/ADC-range budgeting error, not a broken component or wiring mistake.

**8C — Full-chain DC verification, FINAL redesign (Rin1 = 36 kΩ) — VERIFIED, current authoritative result**

Netlist excerpt actually used (`AFE_IntegrationComplete.net`, generated 2026-08-21 15:05:47 from `AFE_IntegrationComplete.asc`):
```
VREF1 N010 0 DC 1.65
Rin1 SUM VAMP 36k
Rref1 SUM N010 30k
Rf3 SUM Vx 30k
Rin2 N009 Vx 10k
Rf4 N009 Vadc 10k
```

| Node | VERIFIED value (Rin1 = 36 kΩ, $V_{sensor}$=+100 mV) |
|---|---:|
| $V_{sensor}$ | +0.100000 V |
| $V_{filter}$ | +0.158993 V |
| $V_{amp}$ | +1.589465 V |
| $V_{ADC}$ | **+2.974337 V** |

Matches the transfer function $V_{ADC}=0.8333\,V_{AMP}+1.65\,\text{V}$ to within 0.22 mV.

| Sensor condition | $V_{filter}$ | $V_{amp}$ | $V_{ADC}$ | Basis |
|---|---:|---:|---:|---|
| $V_{sensor}=-100\,\text{mV}$ | −0.158998 V | −1.58988 V | **0.3251 V** | CALCULATED via VERIFIED transfer function |
| $V_{sensor}=0\,\text{V}$ | −2.244 µV | −0.000206 V | **1.6498 V** | CALCULATED via VERIFIED transfer function |
| $V_{sensor}=+100\,\text{mV}$ | +0.158993 V | +1.589465 V | **2.974337 V** | **VERIFIED — direct LTspice extraction** |

**Result: PASS.** Corrected chain stays within **0.325 V–2.974 V**, comfortably inside 0–3.3 V (margin ≈0.325 V / ≈9.8% at both rails). **This is the current, final, authoritative ADC range for the KiCad design. This is the numeric range that must carry forward — not the Stage 8B −0.735 V/+4.034 V figures.**

**Stage 8 conclusion: COMPLETE.** Key lesson: an ADC has a hard, fixed input window; every upstream stage must be designed *backward* from that constraint (decide the safe ADC window → determine total required gain → only then split it across stages), not forward from "what gain feels reasonable" in each block considered alone. **Hardware validation for Stage 8 has not been performed — all figures use OP1177, not TL082.**

---

## 5. Final Circuit Architecture

```
±100 mV sensor (ASSUMED)
   │
   ▼
Sallen-Key 2nd-order LPF, ~1 kHz cutoff
  R1=R2=1.5 kΩ (CALCULATED, unverified in sim) or 1.6 kΩ (VERIFIED)
  C1=C2=100 nF, Rf1=5.9 kΩ, Rg1=10 kΩ, K=1.59
   │  Vfilter ≈ ±159 mV @ DC/near-DC (VERIFIED, 1.6 kΩ build)
   ▼
Non-inverting amplifier, 10×
  Rf2=90 kΩ, Rg2=10 kΩ
   │  Vamp ≈ ±1.59 V (VERIFIED)
   ▼
Level shifter (2× inverting op-amp stages)
  Rin1=36 kΩ, Rref1=30 kΩ, Rf3=30 kΩ, Rin2=10 kΩ, Rf4=10 kΩ
  VREF=1.65 V (ideal source in sim; real reference circuit NOT VERIFIED)
   │  Vadc = 0.8333·Vamp + 1.65 V  →  0.325 V – 2.974 V (VERIFIED)
   ▼
ADS1115 16-bit ADC (I²C) — NOT VERIFIED in hardware
   │
   ▼
ESP32 (exact board variant NOT VERIFIED)
```

**Op-amps used in every simulation above: OP1177 (×4).** **Op-amp intended for the physical PCB: TL082.** These are not the same part and TL082 has never been simulated in this project — see Section 7.5.

---

## 6. Filter Design (Stage 5, final)

### 6.1 Final topology and values

| Component | Original (VERIFIED) | Current PCB target |
|---|---:|---:|
| R1, R2 | 1.6 kΩ | **1.5 kΩ** (CALCULATED substitution) |
| C1, C2 | 100 nF | 100 nF (filter capacitors — to ground / feedback, not decoupling) |
| Rf1 | 5.9 kΩ | 5.9 kΩ (unchanged) |
| Rg1 | 10 kΩ | 10 kΩ (unchanged) |
| Op-amp | OP1177 (sim) | TL082 (physical, NOT VERIFIED) |
| Supply | ±12 V | ±12 V |

### 6.2 The 1.6 kΩ → 1.5 kΩ substitution

**Trigger:** 1.6 kΩ could not be sourced locally. Documented analysis (2026-08-21) evaluated whether a single 1.5 kΩ, 1% resistor can directly replace both R1 and R2.

| Quantity | 1.6 kΩ (original) | 1.5 kΩ (proposed/final) | Status |
|---|---:|---:|---|
| Resistor deviation | — | 6.25% | CALCULATED |
| Theoretical cutoff $f_c=\frac{1}{2\pi RC}$ | ≈994.72 Hz | ≈1061.03 Hz | CALCULATED |
| LTspice AC-sweep cutoff | ≈997.38 Hz (VERIFIED) | ≈1063.9 Hz | **CALCULATED, NOT VERIFIED by simulation** — see caveat below |
| Q | 0.7092 | 0.7092 (unchanged) | CALCULATED — Q depends only on K=Rf1/Rg1, not on R1/R2 in an equal-component Sallen-Key design |
| Passband gain K | 1.59× | 1.59× (unchanged) | CALCULATED |

**Important caveat on the 1063.9 Hz figure:** dedicated LTspice schematics were prepared for this test (`AFE_SallenKey_R1p5k_test.asc`, `AFE_SallenKey_R1p5k_DCstep_test.asc`, plus 1.6 kΩ baseline-check twins for a same-methodology sanity check). **Confirmed by file inspection: no `.log` or `.raw` output exists for any of these four schematics** — the headless batch simulation never completed in the working environment (reproduced even on the unmodified baseline file, so this is an environment limitation, not a circuit problem). The 1063.9 Hz figure is a calculated extrapolation (applying the same +0.27% real-op-amp correction already observed between the 994.7 Hz theoretical and 997.38 Hz VERIFIED 1.6 kΩ baseline), **not a confirmed simulation result.**

**Action still open:** open `AFE_SallenKey_R1p5k_test.asc` and `AFE_SallenKey_R1p5k_DCstep_test.asc` in LTspice interactively and press Run to get the actual VERIFIED number before treating the ≈1064 Hz figure as fact. This takes seconds but has not yet been done.

**Conclusion (CALCULATED, not yet confirmed by simulation): a single 1.5 kΩ, 1% resistor can replace both R1 and R2 without changing amplifier gain or ADC scaling** — gain (K), Q, level-shifting, and worst-case ADC excursion are all unaffected because they depend on different resistors (Rf1/Rg1, Rf2/Rg2, Rin1/Rref1/Rf3/Rin2/Rf4). The only parameter that moves is the filter cutoff, which is irrelevant to every DC/near-DC verification test already run (the sensor signal of interest sits far below both 997 Hz and ≈1064 Hz).

**1.5 kΩ is the CURRENT intended PCB value for R1/R2.**

### 6.3 Filter capacitors vs. supply-decoupling capacitors

| Capacitor | Value | Role | Type |
|---|---:|---|---|
| C1, C2 | 100 nF | Sets filter cutoff frequency (Sallen-Key network) | **Filter capacitor — signal path, not decoupling** |
| Regulator input caps | 0.33 µF, ceramic | Per typical 78xx/79xx datasheet recommendation | Supply decoupling |
| Regulator output caps | 0.1 µF, ceramic | Per typical 78xx/79xx datasheet recommendation | Supply decoupling |
| Bulk electrolytic | 10–100 µF | Regulator bulk filtering | Supply decoupling |

C1/C2 must never be confused with or substituted by the 100 nF decoupling caps recommended near IC supply pins (Section 16) — they serve entirely different circuit functions even though the value is coincidentally the same.

---

## 7. Amplifier Design (Stage 6, final)

### 7.1 Topology and gain

Non-inverting amplifier, gain Av = 10×. Note: the filter stage upstream already applies its own passband gain K = 1.59×, so the ratio $V_{amp}/V_{filter}$ measured across the amplifier alone (not the whole chain) is what confirms the 10× design target. For the full ±100 mV sensor test chain:

| Node | Value (VERIFIED) |
|---|---:|
| $V_{filter}$ | ≈±159 mV |
| $V_{amp}$ | ≈±1.59 V |
| Effective gain $V_{amp}/V_{filter}$ | ≈9.997× (target 10×, error ≈0.03%) |

### 7.2 Resistor network

| Component | Value |
|---|---:|
| Rf2 (feedback) | 90 kΩ |
| Rg2 (ground leg) | 10 kΩ |

$$A_v=1+\frac{R_{f2}}{R_{g2}}=1+\frac{90\text{k}}{10\text{k}}=10$$

### 7.3 Op-amp: simulation vs. physical build

| | Device |
|---|---|
| **LTspice verification device (all Stage 5–9 results in this document)** | **OP1177** |
| **Physical PCB implementation (intended)** | **TL082** |

OP1177 was used throughout because it was the model on hand for simulation; the physical build targets TL082 for cost/availability. **TL082 has zero LTspice runs in this project's history.** Do not treat any gain, bandwidth, or headroom figure in this document as applying to TL082 hardware.

### 7.4 What must be re-verified with TL082 before final PCB fabrication

| Item | Why it matters | Status |
|---|---|---|
| Supply voltage compatibility | TL082 datasheet max supply must be checked against ±12 V (typically fine for TL082, but must be confirmed against the exact datasheet, not assumed) | NOT VERIFIED |
| Input common-mode range | Must include the signal swing seen at each op-amp's inputs across the whole chain | NOT VERIFIED |
| Output swing | TL082 output swing near ±12 V rails is a JFET-input part with different headroom behavior than the precision bipolar OP1177 — the Stage 6 "±10 V test, ≈2 V headroom" result does NOT carry over | NOT VERIFIED |
| Gain behavior | Filter K=1.59, amplifier Av=10, level-shifter gain=0.8333 all assumed ideal-op-amp behavior; TL082's finite open-loop gain/offset must be re-checked | NOT VERIFIED |
| Behavior around the 1.65 V reference | TL082 input offset voltage and bias current are different from OP1177's — re-check DC accuracy at the level-shifter summing node | NOT VERIFIED |
| Ability to handle the bipolar ±100 mV–±1.6 V signal swings seen at each stage | JFET-input TL082 has different noise/offset/drift characteristics than the precision OP1177 | NOT VERIFIED |
| Interaction with the filter (Stage 5 topology) | TL082's slew rate and bandwidth are different from OP1177's — re-run the Stage 5 AC/transient tests with TL082 | NOT VERIFIED |
| Interaction with the level-shifting stage (Stage 7 topology) | Two cascaded inverting stages accumulate any single-stage offset/gain error — re-run Stage 8C with TL082 before trusting the 0.325–2.974 V window | NOT VERIFIED |

**This is the single most important open item before schematic finalization: re-simulate the complete chain (Stages 5–8) in LTspice with the TL082 SPICE model before ordering PCB-quantity op-amps.**

---

## 8. Level-Shifter Design (Stage 7, final)

### 8.1 Purpose

Convert the bipolar amplified signal into a voltage suitable for the 0–3.3 V ADC, centered on a 1.65 V reference.

### 8.2 Relationship between Vamp, VREF, and Vadc

$$V_{ADC} = 0.8333\times V_{AMP} + 1.65\,\text{V}$$

Intended behavior:
- At zero input: $V_{ADC}\approx1.65\,\text{V}$
- At positive maximum input: $V_{ADC}$ rises above 1.65 V
- At negative maximum input: $V_{ADC}$ falls below 1.65 V

### 8.3 Two different design points — do not confuse them

**SUPERSEDED (original design, Rin1 = 20 kΩ) — VERIFIED by LTspice at the time, but the design itself was later rejected:**

| $V_{sensor}$ | $V_{ADC}$ |
|---|---:|
| 0 V | ≈1.64957 V |
| +100 mV | ≈+4.03384 V — **outside 0–3.3 V** |
| −100 mV | ≈−0.734703 V — **outside 0–3.3 V, below ground** |

**FINAL (current, Rin1 = 36 kΩ) — VERIFIED, this is the design to use:**

| $V_{sensor}$ | $V_{ADC}$ |
|---|---:|
| −100 mV | 0.325 V |
| 0 mV | 1.650 V |
| +100 mV | 2.975 V |

**Do not use the −0.735 V / +4.034 V figures as the final design — they describe a rejected, superseded circuit (Rin1 = 20 kΩ), preserved here only so the design-error story stays traceable.** The current, LTspice-VERIFIED design uses **Rin1 = 36 kΩ**, giving a final ADC range of **0.325 V ≤ Vadc ≤ 2.975 V**, with ≈1.65 V at zero input.

### 8.4 Resistor network (final)

| Component | Value | Role |
|---|---:|---|
| Rin1 | **36 kΩ** | Signal input resistor, summing node (changed from 20 kΩ) |
| Rref1 | 30 kΩ | Reference input resistor |
| Rf3 | 30 kΩ | Feedback, summing stage |
| Rin2 | 10 kΩ | Input resistor, unity-inverter stage |
| Rf4 | 10 kΩ | Feedback, unity-inverter stage |
| VREF | 1.65 V | Reference (ideal source in sim — physical circuit NOT VERIFIED, Section 9) |

### 8.5 Worst-case 1% tolerance stack-up (CALCULATED)

| Term | Nominal | Worst case |
|---|---:|---:|
| $A_{filter}$ | 1.59 | 1.559–1.622 |
| $A_{amplifier}$ | 10.0 | 9.80–10.20 |
| $G_{LS,signal}$ | 0.8333 | 0.8167–0.8500 |
| $G_{LS,offset}$ | 1.000 | 0.98–1.02 |

Pessimistic combined worst case: $V_{ADC,max}\approx3.151\,\text{V}$ (margin 0.149 V), $V_{ADC,min}\approx0.150\,\text{V}$ (margin 0.150 V). Both margins stay positive even in this all-resistors-worst-case-simultaneously scenario (statistically unlikely). **Conclusion: 1% resistors are sufficient; 5% resistors are not recommended** — they would triple the per-ratio error and erode margin once op-amp offset and reference tolerance are added.

**Important:** this stack-up analysis was performed for the OP1177 SPICE model. It does not include TL082's own offset/gain tolerances, which have not yet been characterized in this design (Section 7.5).

---

## 9. 1.65 V Reference

**Why ≈1.65 V:** to place the bipolar amplified signal at the middle of the 3.3 V ADC domain, so the signal has equal headroom swinging positive and negative.

**Proposed generation method:**
```
ESP32 3.3 V
    ↓
Voltage divider
    ↓
≈1.65 V
```

**Current status: this reference has never been simulated as a real circuit.** In every LTspice run to date (Stages 7–9), VREF1 is an **ideal DC voltage source** — no divider, no source impedance, no buffering, no noise. This is explicitly flagged as TBD in the original documentation and remains unresolved.

**Buffered or unbuffered?** The current design does not specify — it has never been drawn as a real divider circuit at all. This must be treated as an open PCB/design decision, not silently resolved by this document. A simple resistor divider off 3.3 V has non-trivial source impedance that feeds directly into Rref1 (30 kΩ) at the summing node; whether that source impedance is low enough to be treated as an ideal source, or whether a low-impedance buffer (e.g., an op-amp voltage follower, or a dedicated reference IC) is required, has **not been analyzed or simulated.**

**Recommendation for a low-noise, reasonably low-impedance reference:** treat this as a design task still to be done — do not simply wire a bare divider into the summing node without first checking (by simulation or by hand calculation of loading error) that its output impedance doesn't shift the 1.65 V offset by more than the tolerance budget already assumed in Section 8.5.

**Status: NOT VERIFIED. This must be designed and simulated before the reference hardware is finalized in KiCad.**

---

## 10. ADC / ESP32 Interface

### 10.1 Options considered

1. **ESP32 internal ADC** — considered, not the current physical design direction.
2. **ADS1115 external ADC** — the current physical design consideration.

### 10.2 ADS1115 — design intention (NOT VERIFIED in hardware)

| Property | Value |
|---|---|
| Resolution | 16-bit |
| Interface | I²C |
| Inputs | 4 single-ended, or 2 differential |
| Supply range | Datasheet-specified; project intends VDD = 3.3 V |
| PGA setting (intended) | ±4.096 V (GAIN_ONE) recommended for the 0–3.3 V signal window |

### 10.3 Connection to ESP32 (intended)

| ADS1115 pin | ESP32 pin | Notes |
|---|---|---|
| SDA | I²C SDA GPIO | Exact GPIO depends on board variant — NOT VERIFIED (Section 15) |
| SCL | I²C SCL GPIO | Same caveat |
| VDD | 3.3 V | — |
| GND | GND | Common ground with analog section — see Section 12 |
| ADDR | Strap to GND/VDD/SDA/SCL per desired I²C address | Address configuration NOT VERIFIED / not yet decided |

**Separation of design intention from verified testing: the ADS1115 has NOT been physically tested with this AFE.** No I²C communication, no ADC reading, no code has been run against real hardware. Everything above is a connection plan, not a verified interface.

---

## 11. Power Supply

### 11.1 Required rails

| Rail | Consumer | Status |
|---|---|---|
| +12 V | Op-amps (all 4 stages) | Intended, not yet built |
| −12 V | Op-amps (all 4 stages) | Intended, not yet built — generation method unresolved, see 11.3 |
| +3.3 V | ESP32, ADS1115 | ESP32-provided |
| ≈+1.65 V | Level-shifter reference | NOT VERIFIED circuit (Section 9) |

The project does **not** currently assume a laboratory bench supply will be available for the final build — a converter/inverter solution is required to generate ±12 V from a lower input (USB 5 V, per the original power-path plan).

### 11.2 Planned power path

```
USB 5 V
  ↓
DC-DC boost (MT3608) — raw rail, positive only
  ↓
Positive linear regulator (7812-type) → clean +12 V
  ↓
[separate negative-rail generation — UNRESOLVED, see 11.3]
  ↓
Negative linear regulator (7912-type) → clean −12 V
```

### 11.3 The MT3608 negative-rail error — explicitly corrected here

**The MT3608 is a boost converter. A boost converter only steps a positive input voltage up to a higher positive output voltage — it cannot generate a negative rail.** Earlier project material (the BOM/shopping lists) listed "MT3608 boost module, qty 1–2" and a power-path diagram showing a single DC-DC stage feeding "raw ±14V," which implicitly treated the MT3608 as capable of producing both rails. **That implication is incorrect and is explicitly superseded here.**

A real negative rail requires one of:
- A dedicated inverting DC-DC converter/module (e.g., a charge-pump or inverting buck-boost), **or**
- A boost converter feeding a separate inverting topology, **or**
- A purpose-built dual-output (±) DC-DC module.

**No specific module has been selected or verified for this role.** The original materials document (Section 03) already correctly flagged "Negative-voltage DC-DC converter (exact module/topology)" as TBD in its "must remain TBD" table — this document confirms that flag is still open and clarifies that MT3608 alone does not resolve it.

### 11.4 Current approved power architecture

| Item | Status |
|---|---|
| +12 V generation (MT3608 boost → 7812 linear reg) | ASSUMED architecture, not yet built or verified |
| −12 V generation | **NOT VERIFIED / NOT RESOLVED** — MT3608 cannot do this alone; a real negative-rail solution must be selected before PCB layout |
| +3.3 V | Provided by ESP32 board | ASSUMED sufficient for ADS1115 |
| 1.65 V reference | NOT VERIFIED (Section 9) |
| Ripple/noise on switching stage | NOT VERIFIED — no oscilloscope has been used to characterize this; the linear regulation stage is expected to attenuate it but this is unverified |

**This is a hard blocker for PCB layout of the power section — do not finalize the power schematic in KiCad until a specific negative-rail component/module is selected and at minimum datasheet-checked (ideally bench-tested).**

---

## 12. Component Selection — Rationale Summary

- **Filter (R1/R2):** 1.5 kΩ chosen over 1.6 kΩ purely for sourcing; CALCULATED safe, not yet LTspice-confirmed (Section 6.2).
- **Level shifter (Rin1):** 36 kΩ is the corrected, LTspice-VERIFIED value; it is a standard E24, 1% value, commonly stocked in generic kits — no exotic value needed.
- **Op-amp:** TL082 chosen over OP1177 for sourcing/cost; **not yet verified in any simulation** — highest-priority open item (Section 7.5).
- **ADC:** ADS1115 chosen for 16-bit resolution and simple I²C interface; not yet hardware-tested with this AFE.
- **Negative rail:** unresolved; MT3608 alone is insufficient (Section 11.3).

---

## 13. Final BOM

| Component | Ref Des | Value / Part Number | Qty | Purpose | Form Factor | Status | Notes |
|---|---|---|---:|---|---|---|---|
| Op-amp | U1–U4 | **TL082** | 2 (dual, 2 pkgs) or 4 (single) | All 4 analog stages | DIP-8/SOIC-8 | **NOT VERIFIED** | Never simulated in this project — re-verify per Section 7.5 before finalizing footprint/qty |
| Filter resistor | R1, R2 | **1.5 kΩ, 1%** | 2 | Sallen-Key frequency-setting | THT/SMD 1% metal film | CALCULATED (sim not run) | See Section 6.2 caveat |
| Filter feedback resistor | Rf1 | 5.9 kΩ, 1% | 1 | Sets filter passband gain K | 1% metal film | VERIFIED | — |
| Filter gain resistor | Rg1 | 10 kΩ, 1% | 1 | Sets filter passband gain K | 1% metal film | VERIFIED | Same value reused elsewhere — buy several |
| Amplifier feedback resistor | Rf2 | 90 kΩ, 1% | 1 | Sets 10× amplifier gain | 1% metal film | VERIFIED | — |
| Amplifier gain resistor | Rg2 | 10 kΩ, 1% | 1 | Sets 10× amplifier gain | 1% metal film | VERIFIED | — |
| Level-shift signal resistor | Rin1 | **36 kΩ, 1%** | 1 | Level-shifter signal gain (corrected value) | 1% metal film, E24 | VERIFIED | Was 20 kΩ — SUPERSEDED, do not use |
| Level-shift reference resistor | Rref1 | 30 kΩ, 1% | 1 | Sets 1.65 V offset ratio | 1% metal film | VERIFIED | — |
| Level-shift feedback resistor | Rf3 | 30 kΩ, 1% | 1 | Summing-stage feedback | 1% metal film | VERIFIED | — |
| Level-shift 2nd-stage resistor | Rin2 | 10 kΩ, 1% | 1 | Unity inverter input | 1% metal film | VERIFIED | — |
| Level-shift 2nd-stage feedback | Rf4 | 10 kΩ, 1% | 1 | Unity inverter feedback | 1% metal film | VERIFIED | — |
| 1.65 V reference divider resistors | Rref_a, Rref_b | TBD — depends on final reference design | 2 | Generate ≈1.65 V from 3.3 V | 1% metal film | **NOT VERIFIED** | Reference circuit not yet designed (Section 9) |
| Filter capacitor | C1, C2 | 100 nF, ceramic | 2 | Sallen-Key frequency-setting | C0G/NP0 preferred | VERIFIED | Signal path, not decoupling |
| Regulator input cap | — | 0.33 µF, ceramic | 2 | Decoupling, regulator input | Ceramic | ASSUMED (datasheet-typical) | Per 78xx/79xx typical application |
| Regulator output cap | — | 0.1 µF, ceramic | 2 | Decoupling, regulator output | Ceramic | ASSUMED (datasheet-typical) | Per 78xx/79xx typical application |
| Bulk electrolytic | — | 10–100 µF | 2–4 | Regulator bulk filtering | Electrolytic, ≥25 V | ASSUMED | — |
| IC decoupling caps | — | 100 nF, ceramic | Per Section 16 table | Local IC supply decoupling | 0402/0603 SMD or THT | ASSUMED (standard practice) | See Section 16 |
| Positive linear regulator | U5 | 7812-type | 1 | +12 V rail | TO-220 | ASSUMED architecture | — |
| Negative linear regulator | U6 | 7912-type | 1 | −12 V rail | TO-220 | ASSUMED architecture | Depends on unresolved negative-rail source, Section 11.3 |
| Boost converter | — | MT3608 | 1 | Steps 5 V → raw positive rail | Module | ASSUMED architecture | **Not** a negative-rail solution — Section 11.3 |
| Negative-rail generator | — | **TBD — unresolved** | — | Generate raw negative rail before 7912 regulation | Module | **NOT VERIFIED / NOT SELECTED** | Hard blocker for power-section layout |
| ADC module | — | ADS1115 | 1 | 16-bit I²C ADC | Breakout module | Design intention only | Never hardware-tested with this AFE |
| MCU | — | ESP32 dev board (exact variant TBD) | 1 | Reads ADC, sends to PC | Dev board | NOT VERIFIED | Exact board dimensions/pinout must be confirmed before footprint creation (Section 18) |
| Connectors | — | TBD | — | Sensor input, power input, programming access | — | NOT VERIFIED | To be defined during layout |
| Pin headers | — | 2.54 mm, M/F | 1 kit | Module interconnects, test points | — | ASSUMED | — |

### Superseded Components (do not use)

| Component | Superseded value | Superseded by | Reason |
|---|---|---|---|
| Op-amp | OP1177 (×4) | TL082 | Sourcing/cost for physical build; TL082 not yet simulated |
| Filter resistor R1/R2 | 1.6 kΩ | 1.5 kΩ | 1.6 kΩ not locally sourceable |
| Level-shift signal resistor Rin1 | 20 kΩ | 36 kΩ | Original value caused ADC range to exceed 0–3.3 V (Stage 8B failure) |
| 100 Ω series-combo workaround (for approximating 1.6 kΩ) | — (dropped) | Direct 1.5 kΩ single resistor | No longer needed once 1.5 kΩ was accepted — simplifies BOM by one value |

---

## 14. PCB Design Architecture

### 14.1 Logical sections

| Section | Contents |
|---|---|
| **A — Sensor input** | Input connector, any input protection (not yet designed) |
| **B — Filter** | Sallen-Key LPF: R1, R2, C1, C2, Rf1, Rg1, U1 |
| **C — Amplifier** | Non-inverting 10× stage: Rf2, Rg2, U2 |
| **D — Level shifter** | Two cascaded inverting stages: Rin1, Rref1, Rf3, Rin2, Rf4, U3, U4, 1.65 V reference divider |
| **E — ADC** | ADS1115 module/footprint, I²C pull-ups (datasheet-typical, value NOT VERIFIED) |
| **F — ESP32** | ESP32 dev board footprint (exact variant TBD, Section 18) |
| **G — Power** | MT3608, negative-rail generator (TBD), 7812, 7912, all decoupling/bulk caps |

### 14.2 Signal flow

Left-to-right layout is strongly recommended: **A → B → C → D → E → F**, with **G (power)** typically along one edge or the bottom, feeding all sections' supply pins directly rather than daisy-chaining through the signal sections. This makes the analog signal path visually traceable on the silkscreen and during debugging.

### 14.3 Layout priorities

- Short analog signal paths, especially B→C→D (high-impedance, precision nodes)
- Clean grounding (Section 15)
- Physical separation of noisy digital circuitry (F — ESP32, I²C) from sensitive analog circuitry (B, C, D)
- Supply decoupling placed immediately at each IC's supply pins (Section 16)
- Connectors placed at board edges for accessibility
- Test points (Section 17) placed for oscilloscope/multimeter probing without desoldering
- Beginner-friendly routing — this is a first PCB; prioritize clarity and debuggability over density

---

## 15. Grounding Strategy

Given this is a first PCB intended to be understandable and robust for a beginner, a single continuous ground plane is recommended rather than a split analog/digital plane — split planes only pay off when there's a specific, quantified noise problem to solve, and introducing one without justification adds a real risk (an accidental gap in the split creating a slot antenna / return-path discontinuity) for no proven benefit at this stage.

**Recommended approach:**
- **One continuous ground plane** across the board, tied to a single point at the power input.
- Treat the following as *ground domains* that share the plane but should be **routed thoughtfully relative to each other**, not electrically separated:
  - Analog ground (Sections B, C, D — filter/amp/level-shifter)
  - ADC ground (Section E)
  - ESP32/digital ground (Section F)
  - Power ground (Section G — regulator and converter returns)
- **Keep noisy return paths away from sensitive analog nodes:** route the MT3608 switching-converter's high-current/high-di/dt return path so it does not cross under or near the filter/amplifier/level-shifter section (B/C/D). Physically locate G (power) so its return current doesn't have to travel through A/B/C/D to get back to the source.
- **ESP32/ADS1115 digital (I²C) traces and switching-regulator traces** should not run parallel to or over the analog section for any significant length.
- A single-point (or short, wide, low-impedance) tie between the "quiet" analog region and the main ground plane is sufficient at this scale — a full star-ground with separate copper pours is not justified for a board this size and is more likely to introduce beginner routing mistakes than to solve a real problem.

This is intentionally a simple, single-plane strategy with careful *placement and routing discipline*, not a complicated multi-plane architecture — appropriate for a first PCB with no oscilloscope-verified noise problem to design against yet.

---

## 16. Decoupling Strategy

| Device | Supply pin | Ground pin | Capacitor value | Placement requirement |
|---|---|---|---|---|
| TL082 (×4, Sections B/C/D) | V+ (+12 V), V− (−12 V) | — | 100 nF ceramic on each supply pin to local ground, per op-amp | As close as physically possible to the IC's supply pins — this is standard practice, not yet datasheet-confirmed for TL082 specifically |
| ADS1115 | VDD (3.3 V) | GND | 100 nF ceramic | Directly at the module's VDD pin |
| ESP32 module | 3.3 V | GND | Per module design (usually already decoupled on-module) | If ESP32 is mounted via headers rather than a raw module, on-board decoupling is likely already present — confirm against the specific board variant (Section 18) |
| 7812 regulator | Input, Output | GND | 0.33 µF (input) / 0.1 µF (output), ceramic | At the regulator pins, per typical 78xx application circuit |
| 7912 regulator | Input, Output | GND | 0.33 µF (input) / 0.1 µF (output), ceramic | At the regulator pins, per typical 79xx application circuit |
| MT3608 boost module | VIN | GND | Per module datasheet — typically pre-populated on the module itself | If using a pre-built breakout module, additional local decoupling on the main board is still recommended at the module's power-in connector |
| Negative-rail generator (TBD) | — | — | TBD | Cannot be specified until the module/topology is selected (Section 11.3) |
| Bulk/reservoir | Regulator inputs (raw rail) | GND | 10–100 µF electrolytic | Near each regulator's input, upstream of the small ceramic caps |

General rule applied throughout: decoupling capacitors must be physically as close as possible to the IC/module's supply and ground pins — this minimizes trace inductance between the capacitor and the device, keeping the decoupling effective at high frequency.

---

## 17. Test Points

| TP | Location | What to measure | Purpose |
|---|---|---|---|
| TP1 | Sensor input | Raw ±100 mV signal (or fake-sensor divider voltage during bring-up) | Confirms the input actually reaching the board matches the intended test condition |
| TP2 | Filter output ($V_{filter}$) | Expect ≈0 V / +159 mV / −159 mV at DC test points (1.5 kΩ or 1.6 kΩ build) | Confirms Stage 5 filter/gain stage before the amplifier |
| TP3 | Amplifier output ($V_{amp}$) | Expect ≈0 V / +1.59 V / −1.59 V | Confirms Stage 6 amplifier gain independent of the level shifter |
| TP4 | 1.65 V reference node | Expect 1.65 V ±1% | Catches a bad divider/reference before it corrupts every downstream measurement — check this *before* wiring into the level shifter, per the original hardware verification plan |
| TP5 | ADC input ($V_{ADC}$) | Expect ≈0.325 V / 1.650 V / 2.975 V at the three DC test conditions | Confirms the complete analog chain before trusting any ADS1115 reading |
| TP6 | +12 V rail | 12 V ±regulator tolerance (e.g., 11.5–12.5 V) | Confirms positive supply before powering op-amps |
| TP7 | −12 V rail | −12 V ±regulator tolerance | Confirms negative supply before powering op-amps — **this rail's generation method is currently unresolved (Section 11.3)**, so this test point is especially important during bring-up |
| TP8 | +3.3 V rail | 3.3 V ±tolerance | Confirms digital/ADC supply |
| TP9 | GND | 0 V reference / continuity check | Common reference for all other measurements |

The board should allow the entire analog signal chain (TP1→TP2→TP3→TP5) to be debugged with a multimeter (and, if available later, an oscilloscope) without desoldering any component.

---

## 18. ESP32 Integration

**Exact ESP32 development-board variant: NOT VERIFIED / not yet identified.** The original materials documentation specifies only "any standard ESP32 dev board" / "any WROOM-32 based board" — this is not sufficient to create a KiCad footprint. Before footprint creation, the following must be explicitly confirmed against the actual physical board that will be used:

| Item to verify | Why |
|---|---|
| Exact board variant (e.g., ESP32 DevKitC V4, NodeMCU-32S, ESP32-WROOM-32 generic dev board, etc.) | Pin count, pin spacing, and pinout differ between variants |
| Module orientation | Determines silkscreen/footprint mirroring |
| Header spacing (0.1"/2.54 mm single or double row, and row-to-row spacing) | Common variants differ (e.g., 2×19 vs. other configurations) |
| Required GPIOs: I²C SDA pin, I²C SCL pin | Board-variant-specific pin labeling |
| 3.3 V and GND pin locations | — |
| Reset/boot button accessibility | Needed for programming without desoldering |
| USB port accessibility | Must remain reachable after the board is seated in headers, for programming/power |
| Antenna clearance | Keep copper pour and tall components clear of the module's antenna area per the specific board's layout |
| Programming/debug access | Confirm whether USB-UART is onboard or an external programmer is needed |

**Do not create the ESP32 footprint in KiCad until the exact board variant is chosen and its dimensions/pinout are confirmed from its actual datasheet/pinout diagram** — assuming a generic "ESP32 dev board" footprint risks a mechanical mismatch.

---

## 19. PCB Size

**Current physical target: ≈100 mm × 100 mm.**

Given Sections A–G above (input, filter, amplifier, level shifter, ADC, ESP32, power), a 100×100 mm board is reasonably sized to fit all sections with room for the layout priorities in Section 14.3 — clear labeling, accessible test points, non-cramped decoupling placement, and beginner-friendly routing — rather than a minimum-area layout. This has not been validated with an actual component placement/routing exercise; it is a reasonable target given the component count, not a confirmed fit.

**Explicitly do not optimize for minimum size.** The first PCB should prioritize easy assembly, easy probing, clear labeling, accessibility, modularity, and troubleshooting over board-area efficiency.

---

## 20. Verified / Unverified Engineering Parameters

| Item | Status | Evidence | Action Required |
|---|---|---|---|
| Filter cutoff, 1.6 kΩ build | VERIFIED | LTspice AC sweep, ≈997.38 Hz | None — build reference only, since PCB uses 1.5 kΩ |
| Filter cutoff, 1.5 kΩ build | **CALCULATED, NOT VERIFIED** | Extrapolated ≈1063.9 Hz; no `.log`/`.raw` exists for the R1p5k schematics | Run `AFE_SallenKey_R1p5k_test.asc` / `..._DCstep_test.asc` in LTspice interactively before treating as final |
| Filter attenuation / roll-off | VERIFIED | Ideal E-source ≈−39.95 dB/decade; real OP1177 full-chain ≈−37.3 dB/decade (1 kHz–10 kHz) | None |
| Amplifier gain (10×) | VERIFIED | DC tests: Av=9.996–10.0001; chain cross-check Av≈9.997 | None (for OP1177) |
| 1.5 kΩ substitution safety (gain/Q/ADC-range unaffected) | CALCULATED | Section 6.2/21 analysis | Confirm with the LTspice run above for completeness |
| Level shifter transfer function (Rin1=36k) | VERIFIED | Full-chain `.op` re-sim, netlist + raw extraction, 2026-08-21 | None (for OP1177) |
| 1.65 V reference circuit | **NOT VERIFIED** | Only ever an ideal DC source in every simulation | Design and simulate a real divider/reference circuit; check loading against Rref1 |
| ADC range (0.325–2.975 V) | VERIFIED | Direct LTspice extraction + confirmed transfer function | None (for OP1177); re-check once TL082 substitution is verified |
| TL082 behavior (all stages) | **NOT VERIFIED** | Never appears in any simulation; only OP1177 was ever run | Re-simulate Stages 5–8 with the TL082 SPICE model before ordering PCB-quantity parts |
| ADS1115 operation with this AFE | **NOT VERIFIED** | Design intention only; no hardware test performed | Bench-test after PCB assembly, or on breadboard first |
| +12 V supply | **ASSUMED architecture** | MT3608 → 7812, not yet built | Build and multimeter-verify before trusting downstream analog stages |
| −12 V supply | **NOT VERIFIED / NOT RESOLVED** | MT3608 is boost-only and cannot generate a negative rail; no negative-rail module has been selected | Select and datasheet/bench-verify a real negative-rail solution before power-section layout |
| ESP32 interface | **NOT VERIFIED** | Exact board variant not identified; I²C wiring is a plan, not a tested connection | Choose exact board variant, confirm pinout, bench-test I²C before layout |
| Sensor interface | ASSUMED | ±100 mV is a design target, never measured from a physical sensor | Characterize actual sensor if/when available; input protection not yet designed |
| PCB dimensions (100×100 mm) | ASSUMED | Stated target, not validated by placement/routing | Validate with an actual component placement pass during layout |
| Connector pinout | **NOT VERIFIED** | Not yet defined | Define during layout (Section 13) |
| Decoupling | ASSUMED (standard practice) | Values are datasheet-typical, not project-specific measurements | Confirm TL082/ADS1115/regulator decoupling recommendations against their actual datasheets |
| Grounding | ASSUMED (design recommendation) | Reasoned recommendation in Section 15, not simulated/measured | Apply during layout; revisit only if a real noise problem is later measured |

---

## 21. Engineering Decisions That Must NOT Be Repeated

- **Do not use the MT3608 as if it were a negative-voltage inverter.** It is a boost converter; it only produces a higher positive voltage. A real negative-rail solution (inverting converter/module) is still unresolved (Section 11.3) — this is a hard blocker for power-section layout, not a solved problem.
- **Do not assume OP1177 will be the physical op-amp.** Every simulated result in this entire project — every gain figure, every dB number, every ADC voltage — was generated with the OP1177 SPICE model. The physical build targets TL082, which has never been simulated. Treat every quantitative result in this document as provisional until re-verified with TL082 (Section 7.5).
- **Do not use 1.6 kΩ simply because it appeared in the original simulation.** The current physical target is 1.5 kΩ, and even that substitution's predicted cutoff frequency has not been confirmed by an actual LTspice run (Section 6.2) — only calculated.
- **Do not assume the 1.65 V reference is automatically perfect** without considering its source impedance and interaction with Rref1 (30 kΩ). It has only ever existed as an ideal DC source in simulation (Section 9).
- **Do not assume an ADC input is safe merely because the theoretical signal is centered around 1.65 V.** The original design (Rin1=20 kΩ) was also centered on 1.65 V at zero input and still swung to −0.735 V / +4.034 V at the extremes — centering alone does not guarantee staying inside 0–3.3 V; the full transfer function and gain budget must be checked (Section 8).
- **Do not fabricate the PCB until all critical power and component-compatibility issues are resolved** — specifically: the negative-rail generation method, TL082 re-verification across the full chain, and the 1.65 V reference circuit design.
- **Do not treat the Stage 8B figures (−0.735 V / +4.034 V) as the final ADC design.** They describe the rejected, superseded original level-shifter design (Rin1=20 kΩ). The final, LTspice-verified design uses Rin1=36 kΩ and yields 0.325 V–2.975 V.
- **Do not silently discard or "fix" superseded values while consolidating documentation.** Every earlier measurement in this document is preserved with an explicit SUPERSEDED marker rather than deleted, per the project's own documentation discipline.

---

## 22. Final PCB Design Checklist

### Before schematic finalization
- [ ] All component values confirmed (R1/R2 = 1.5 kΩ pending LTspice re-confirmation, Section 6.2)
- [ ] TL082 selected **and re-verified** across Stages 5–8 (currently NOT done — Section 7.5)
- [ ] Filter values confirmed (1.5 kΩ cutoff simulation still outstanding)
- [ ] Amplifier gain confirmed (VERIFIED for OP1177; re-check for TL082)
- [ ] Level-shifter values confirmed (Rin1 = 36 kΩ, VERIFIED for OP1177)
- [ ] 1.65 V reference confirmed (currently NOT VERIFIED — real circuit not yet designed)
- [ ] ADC selection confirmed (ADS1115 chosen; hardware not tested)
- [ ] ESP32 board variant confirmed (currently NOT identified — Section 18)
- [ ] +12 V supply confirmed (architecture assumed, not built/measured)
- [ ] −12 V supply confirmed (**currently unresolved** — Section 11.3)
- [ ] Capacitors confirmed (filter caps VERIFIED; decoupling values ASSUMED/typical)
- [ ] Resistor values confirmed (all VERIFIED except R1/R2 at 1.5 kΩ)
- [ ] Connectors confirmed (not yet defined)

### Before PCB layout
- [ ] Correct footprints (cannot be finalized for ESP32/TL082/op-amp package until variants are confirmed)
- [ ] Correct pin numbering
- [ ] Correct polarity (electrolytics, regulators)
- [ ] Correct package orientation
- [ ] Decoupling locations planned (Section 16)
- [ ] Test points added (Section 17, TP1–TP9)
- [ ] Mounting holes considered
- [ ] ESP32 antenna clearance considered (pending board-variant confirmation)

### Before PCB fabrication
- [ ] ERC completed
- [ ] DRC completed
- [ ] No unconnected nets
- [ ] No unintended shorts
- [ ] Power rails verified (including a resolved, real negative-rail source)
- [ ] Connector pinouts verified
- [ ] Silkscreen reviewed
- [ ] Board dimensions verified (100×100 mm target validated against actual placement)
- [ ] Gerbers reviewed
- [ ] Final schematic and PCB cross-checked

---

## 23. Remaining Engineering Tasks (priority order)

1. **Re-simulate Stages 5–8 in LTspice with the TL082 model** instead of OP1177 — this affects every downstream number in the document and is the single highest-priority open item.
2. **Actually run** `AFE_SallenKey_R1p5k_test.asc` and `AFE_SallenKey_R1p5k_DCstep_test.asc` in LTspice (interactive, not headless) to confirm the ≈1064 Hz cutoff and the DC-step measurements, replacing the current calculated extrapolation with a real result.
3. **Design and simulate the 1.65 V reference circuit** as a real divider (and decide buffered vs. unbuffered), then re-check its loading effect on Rref1.
4. **Select and verify a real negative-rail (−12 V) generation solution** — the MT3608 alone does not solve this; datasheet-check (and ideally bench-test) a specific module/topology.
5. **Identify the exact ESP32 development-board variant** to be used, and pull its real pinout/dimensions before creating any KiCad footprint.
6. **Define connector pinouts** for sensor input, power input, and programming/debug access.
7. **Bench-test the ADS1115 with the ESP32** over I²C independent of the analog front end, to de-risk the digital interface before full-board bring-up.
8. **Validate the 100×100 mm board size** against an actual component placement pass once footprints exist.

---

## 24. Consistency Audit

Findings from auditing this consolidation against the source documents:

| Finding | Detail | Resolution applied in this document |
|---|---|---|
| Conflicting R1/R2 values within the source doc itself | Stage 3 (Section 4 of this doc) and the original Section 14 "Final Component Table" both list 1.6 kΩ; the original Section 21 addendum changes it to 1.5 kΩ, and Section 12.2 of the source doc shows both (strikethrough 1.6 kΩ → 1.5 kΩ) | This document uses **1.5 kΩ** as the current PCB target throughout, with 1.6 kΩ explicitly marked SUPERSEDED (Section 6.2, 13) |
| Conflicting Rin1 values | Original Section 8.1 (20 kΩ) vs. Section 8.4/11–13 (36 kΩ) | This document uses **36 kΩ** as final, with 20 kΩ explicitly marked SUPERSEDED (Section 8) |
| Conflicting ADC range figures | Original Section 10.1 (−0.735 V to +4.034 V, Rin1=20k) vs. Section 13 (0.325–2.975 V, Rin1=36k) | This document uses **0.325–2.975 V** as final; the −0.735/+4.034 V figures are preserved only as historical/SUPERSEDED evidence of the Stage 8 failure |
| Op-amp identity conflict (new in this consolidation) | Every source-document simulation used OP1177; this consolidation task introduced TL082 as the physical target, which appears nowhere in prior simulation history | Flagged throughout (Sections 4, 7, 20, 21) as **NOT VERIFIED** — no result in this document should be read as applying to TL082 |
| MT3608 role ambiguity | Materials doc (Section 03) correctly lists the negative converter as TBD, but later BOM/shopping-list sections list "MT3608, qty 1–2" without clarifying it only covers the positive path, and the power-path diagram shows one DC-DC stage feeding "raw ±14V" | Corrected explicitly in Section 11.3 of this document — MT3608 is boost-only; negative rail remains unresolved |
| 1.5 kΩ cutoff frequency presented as if simulated | Original Section 21.6 states the headless batch simulation "did not complete" but Section 21.2/21.3 present ≈1061–1064 Hz figures without a status tag distinguishing calculation from simulation | This document explicitly tags the 1.5 kΩ cutoff as **CALCULATED, NOT VERIFIED** (Section 6.2, 20) and confirmed by direct file-timestamp inspection that no `.log`/`.raw` exists for the R1p5k schematics |
| Missing figure asset | Stage 4 references `Pasted image 20260820120740.png`, which does not exist anywhere in the project directory | Flagged in Section 4 (Stage 4) rather than fabricated or silently omitted |
| Values appearing in BOM but not yet in any schematic | 1.65 V reference divider resistors, negative-rail generator components, connectors | Listed in Section 13 BOM with explicit NOT VERIFIED / TBD status rather than fabricated values |
| No values found appearing in a schematic but missing from the BOM | — | None found — the LTspice netlists and the original BOM tables use the same resistor/capacitor value set (once the 1.5 kΩ/36 kΩ corrections are applied) |
| No conflicting capacitor values found | C1/C2=100 nF is consistent across every source reference | No action needed |
| No conflicting supply voltage found | ±12 V for op-amps and 3.3 V for digital are consistent across every source reference | No action needed |

---

## 25. FINAL DESIGN — DO NOT CHANGE WITHOUT RE-VERIFICATION

This is the exact circuit to transfer into KiCad. Any deviation from these values requires a fresh LTspice run before being treated as safe.

**Filter (Stage 5):**
R1 = R2 = 1.5 kΩ *(CALCULATED substitution — confirm with an actual LTspice run before ordering in quantity, Section 6.2)*
C1 = C2 = 100 nF
Rf1 = 5.9 kΩ, Rg1 = 10 kΩ (K = 1.59, Q ≈ 0.709)

**Amplifier (Stage 6):**
Rf2 = 90 kΩ, Rg2 = 10 kΩ (Av = 10)

**Level shifter (Stage 7):**
Rin1 = 36 kΩ *(corrected from 20 kΩ — VERIFIED)*
Rref1 = 30 kΩ
Rf3 = 30 kΩ
Rin2 = 10 kΩ
Rf4 = 10 kΩ
VREF = 1.65 V *(real circuit NOT VERIFIED — Section 9)*
Transfer function: $V_{ADC}=0.8333\,V_{AMP}+1.65\,\text{V}$

**Resulting ADC range (VERIFIED, OP1177 model):**
$V_{sensor}=-100\,\text{mV}\rightarrow V_{ADC}\approx0.325\,\text{V}$
$V_{sensor}=0\,\text{mV}\rightarrow V_{ADC}\approx1.650\,\text{V}$
$V_{sensor}=+100\,\text{mV}\rightarrow V_{ADC}\approx2.975\,\text{V}$

**Op-amp:**
Simulation device: OP1177 (all results above)
Physical build target: **TL082 — NOT YET VERIFIED in any simulation. Re-simulate before fabrication.**

**Power:**
+12 V, −12 V (generation method for −12 V unresolved — Section 11.3), +3.3 V (ESP32-provided), ≈1.65 V reference (circuit not yet designed — Section 9)

**Digital interface:**
ADS1115 (I²C, 3.3 V, PGA ±4.096 V intended) → ESP32 (exact board variant unresolved — Section 18)

**PCB target:** ≈100 mm × 100 mm, sections A–G laid out left-to-right per Section 14.

**Open blockers before fabrication (must be resolved, not carried forward silently):**
1. TL082 full-chain re-simulation
2. Real 1.65 V reference circuit design + simulation
3. Real negative-rail (−12 V) generation module selection
4. Exact ESP32 board variant identification
5. Confirmatory LTspice run for the 1.5 kΩ filter cutoff
