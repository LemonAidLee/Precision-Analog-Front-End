# 04 — Analog Circuit Design

[← Documentation index](README.md)

Every component value in this document reflects the finalized KiCad schematic (`Precision_AFE_ESP32.kicad_sch`, Rev A). Where earlier iterations provided foundational insights, they are noted to illustrate the iterative engineering process.

---

## Component summary — analog chain

| Ref | Value | Stage | Role |
|---|---|---|---|
| R1, R2 | 1.5 kΩ | 1 | Sallen-Key frequency-setting resistors |
| C1, C2 | 100 nF | 1 | Sallen-Key frequency-setting capacitors |
| Rf1 | 5.9 kΩ | 1 | Feedback — precisely sets passband gain K, and therefore Q |
| Rg1 | 10 kΩ | 1 | Gain-setting leg to ground |
| Rf2 | 90 kΩ | 2 | Feedback — sets precise ×10 amplification |
| Rg2 | 10 kΩ | 2 | Gain-setting leg to ground |
| Rin1 | 36 kΩ | 3 | Signal input to summing node |
| Rref1 | 30 kΩ | 3 | Reference input to summing node |
| Rf3 | 30 kΩ | 3 | Summing-amplifier feedback |
| Rin2 | 10 kΩ | 4 | Inverter input |
| Rf4 | 10 kΩ | 4 | Inverter feedback |
| Rref_a1, Rref_b1 | 470 Ω | ref | 3.3 V → 1.65 V precision divider |
| C3 | 100 nF | ref | Reference bypass for noise suppression |
| U1, U2 | TL082 | all | Two dual JFET-input precision op-amps, DIP-8 |

---

<a id="stage-1"></a>

## Stage 1 — Sallen-Key low-pass filter (U1A)

### Purpose
Band-limit the incoming sensor signal before applying main amplification, ensuring that out-of-band interference is thoroughly suppressed rather than amplified alongside the signal.

### Topology
Unity-gain-plus Sallen-Key (VCVS) low-pass, leveraging an equal-component design. The op-amp operates as a non-inverting amplifier of gain K; R1/R2/C1 establish the forward path, while C2 utilizes positive feedback from the output to dynamically shape the response.

```
 VSENSOR ──[R1 1.5k]──┬──[R2 1.5k]──┬────────┐
                      │             │        │  +
                    [C2]          [C1]       ├──▶╲
                    100n          100n       │    ╲___ VFILTER
                      │             │     ┌──┤    ╱   │
                      │            GND    │  ├──▶╱    │
                      │                   │  │  −     │
                      └───────────────────│──│────────┘
                                          │  │
                                     [Rg1 10k]│
                                          │  └──[Rf1 5.9k]──┘
                                         GND
```

*(U1 pin 3 = +IN, pin 2 = −IN, pin 1 = OUT.)*

### Equations

With `R1 = R2 = R` and `C1 = C2 = C`:

```
             K · ω₀²
H(s) = ─────────────────────
       s² + (ω₀/Q)s + ω₀²

K   = 1 + Rf1/Rg1
ω₀  = 1/(R·C)
Q   = 1/(3 − K)
```

### Evaluated with the actual values

```
K   = 1 + 5.9 kΩ / 10 kΩ            = 1.590          (4.03 dB)
f₀  = 1 / (2π · 1.5 kΩ · 100 nF)    = 1061.0 Hz
Q   = 1 / (3 − 1.590)               = 0.7092
ζ   = 1/(2Q)                        = 0.705
```

### Why K = 1.59 — the key design constraint

In an equal-component Sallen-Key architecture, **the passband gain and the pole Q are deeply interdependent**: `Q = 1/(3 − K)`. Selecting a gain value implicitly determines the filter damping.

| K | Q | Response shape |
|---|---|---|
| 1.000 (unity buffer) | 0.500 | Overdamped — soft, early knee; −3 dB well below f₀ |
| **1.586** | **0.7071** | **Butterworth — maximally flat passband** |
| 1.590 (this design) | 0.7092 | 0.3 % from Butterworth; no measurable peaking |
| 2.000 | 1.000 | 1.25 dB of passband peaking |
| 3.000 | ∞ | Oscillator |

The mathematically ideal Butterworth value is K = 1.5858, necessitating an Rf1/Rg1 ratio of 0.5858. By selecting **5.9 kΩ / 10 kΩ**, the design achieves K = 1.590, representing a negligible 0.25 % error in gain and a 0.30 % error in Q, all while utilizing easily sourceable E96 components. This delivers a superbly flat passband ideal for precision data acquisition.

The 1.59× gain introduced by this stage is not arbitrary—it is a deliberate, highly calculated component of the overall signal budget.

### Relationship to the next stage
The op-amp provides a robust, low-impedance output, allowing it to easily drive Stage 2 without loading effects. By establishing filtering as the first priority, the architecture inherently rejects noise before the main amplification stage magnifies it.

### Verification status
- Topology and equations: **Calculated and validated**
- AC response and roll-off: **Simulated with precision OP1177 models** at R = 1.6 kΩ, confirming ideal behavior.
- AC response at R = 1.5 kΩ with TL082: **Scheduled** for the next simulation phase.
- Physical validation: **Planned for the next development stage.**

---

## Stage 2 — Non-inverting amplifier, ×10 (U1B)

### Purpose
Effectively scale the signal from ±159 mV to ±1.59 V, optimizing the dynamic range presented to the ADC.

### Topology
A precision non-inverting amplifier configuration. VFILTER drives the high-impedance U1 pin 5 (+IN of unit B); Rf2 and Rg2 elegantly set the gain.

### Equation

```
G₂ = 1 + Rf2/Rg2 = 1 + 90 kΩ / 10 kΩ = 10.00      (20.00 dB)
```

### Why non-inverting
Implementing a non-inverting stage ensures the input impedance is defined by the op-amp's intrinsic characteristics. For the TL082's JFET inputs, this presents a near-infinite impedance, perfectly preserving the filter's transfer function without loading it. 

### Voltage range and headroom

| | Value |
|---|---|
| Input | ±159 mV |
| Output | ±1.590 V (3.18 V peak-to-peak) |
| Supply rails | ±12 V |
| Output headroom | ≈ 10.4 V on each rail |

Operating on ±12 V rails provides this stage with phenomenal headroom, guaranteeing completely linear operation without any risk of clipping.

### Bandwidth
The closed-loop bandwidth is governed by GBW / G₂. Simulation confirms the response remains flat to within 0.1 dB up to ~10 kHz, ensuring this gain block introduces absolutely no shaping to the established passband.

### Relationship to the next stage
This stage readies a robust, bipolar ±1.59 V signal for Stage 3, which handles the complex tasks of level shifting and scaling for the single-supply ADC.

---

## Stage 3 — Inverting summing amplifier / level shifter (U2A)

### Purpose
Transform the bipolar signal into a unipolar format centered precisely on a 1.65 V reference, while applying calculated scaling to keep the signal comfortably within the 0–3.3 V ADC window.

### Topology
An inverting summing amplifier designed for perfect signal decoupling.

```
  VAMP ──[Rin1 36k]──┬──────────────────┐
                     │                  │
  VREF ──[Rref1 30k]─┤        −         │
                     ├──────────▶╲      │
                    GND ─────────▶╱─────┴──▶ VX
                                +  │
                     └──[Rf3 30k]──┘
```

### Equation

```
VX = −(Rf3/Rin1)·VAMP  −  (Rf3/Rref1)·VREF
   = −(30k/36k)·VAMP   −  (30k/30k)·VREF
   = −0.8333·VAMP      −  1.000·VREF
```

### Engineering choice: Summing amplifier vs. passive offset
By summing currents at a **virtual ground** (held at ≈ 0 V by the op-amp), the signal and reference paths operate completely independently. Adjusting the signal gain via Rin1 has absolutely zero impact on the reference injection ratio, creating a highly modular and analysable circuit. A passive resistive network would inextricably link these variables, making tuning difficult.

### Why Rin1 = 36 kΩ — the headroom decision

Selecting Rin1 = 36 kΩ is one of the project's most sophisticated design decisions. An initial study used **Rin1 = 20 kΩ** (Gain = 1.5):

| Rin1 | Signal gain | VADC at −100 mV | VADC at +100 mV | Design Safety |
|---|---:|---:|---:|---|
| 20 kΩ (superseded) | 1.500 | −0.735 V | +4.034 V | Output clips against both 0 V and 3.3 V rails |
| 30 kΩ | 1.000 | +0.06 V | +3.24 V | Uncomfortably tight ~60 mV margin |
| **36 kΩ (current)** | **0.8333** | **+0.313 V** | **+2.962 V** | **Excellent ~300 mV safety margin on both rails** |

This choice powerfully demonstrates that centering a signal is only half the problem—the full dynamic span must be rigorously calculated against the supply boundaries. Selecting 36 kΩ sacrifices a small amount of gain to guarantee signal integrity across temperature variations, component tolerances, and supply drift.

### Voltage range

| VSENSOR | VAMP | VX |
|---:|---:|---:|
| −100 mV | −1.590 V | −0.313 V |
| 0 | 0 V | −1.637 V |
| +100 mV | +1.590 V | −2.962 V |

The ±12 V power architecture is critical here, as VX operates entirely in the negative voltage domain, a feat impossible with a single-supply op-amp.

---

## Stage 4 — Unity-gain inverter (U2B)

### Purpose
Restore the signal to its true polarity and drive the ADC's switched-capacitor inputs from a low-impedance op-amp output.

### Topology and equation

```
VADC = −(Rf4/Rin2)·VX = −(10 kΩ/10 kΩ)·VX = −VX

∴  VADC = 0.8333·VAMP + VREF = 13.25·VSENSOR + VREF
```

### Design elegance
While earlier stages could conceptually be reordered or inverted, doing so compromises impedance matching and mathematical decoupling. By utilizing the second half of the U2 dual op-amp package, the design elegantly restores polarity and buffers the signal for the ADC at the cost of only two passive resistors.

### Tolerance tracking
Because Rin2 and Rf4 are identical 10 kΩ resistors sourced from the same reel, their temperature coefficients and manufacturing tolerances will closely track each other. This results in a gain remarkably close to exactly −1, even when using standard 1 % components.

---

## Reference generator

```
+3.3 V ──[Rref_a1 470Ω]──┬── VREF ──▶ Rref1 (30 kΩ, to U2A summing node)
                         │
                       [C3 100n]
                         │
                  [Rref_b1 470Ω]
                         │
                        GND
```

| Quantity | Value |
|---|---|
| Ideal output | 3.3 V × 470/(470+470) = **1.650 V** |
| Thévenin source impedance | 470 ∥ 470 = **235 Ω** |
| Load current into Rref1 | 1.65 V / 30 kΩ ≈ **55 µA** |
| Loading effect | 55 µA × 235 Ω ≈ **−12.9 mV** |
| **Effective VREF** | **≈ 1.637 V** (Confirmed in LTspice: 1.6372 V) |

**Engineering intent:** Generating the reference directly from the ADC's 3.3 V supply rail establishes a robust ratiometric architecture. I specifically selected low-value 470 Ω resistors to minimize the Thévenin source impedance to just 235 Ω. This smartly limits the inevitable loading error from the summing amplifier to a mere 13 mV. This predictable, constant offset is easily accounted for in firmware, avoiding the need for an expensive dedicated voltage reference IC while achieving excellent stability.

---

## Overall transfer function

```
VADC = (1 + Rf1/Rg1) · (1 + Rf2/Rg2) · (Rf3/Rin1) · (Rf4/Rin2) · VSENSOR  +  (Rf3/Rref1)·(Rf4/Rin2)·VREF

     = 1.590 · 10.00 · 0.8333 · 1.000 · VSENSOR  +  1.000 · VREF

     = 13.25 · VSENSOR + VREF
```

---

## Error budget

### Resistor tolerance stack-up (calculated, 1 % parts, worst-case scenario)

| Term | Nominal | Worst-case high | Deviation |
|---|---:|---:|---:|
| 1 + Rf1/Rg1 | 1.590 | 1.6019 | +0.75 % |
| 1 + Rf2/Rg2 | 10.00 | 10.182 | +1.82 % |
| Rf3/Rin1 | 0.8333 | 0.8502 | +2.02 % |
| Rf4/Rin2 | 1.000 | 1.0202 | +2.02 % |
| **Total gain** | **13.25** | **14.15** | **+6.8 %** |

Applying the same rigorous stack-up analysis to VREF (resulting in a loaded range of 1.633 … 1.653 V):

| Worst-case combination | VADC Limit |
|---|---:|
| Max gain + max VREF, at VSENSOR = +100 mV | **3.068 V** — Maintains a safe 232 mV margin below the 3.3 V rail |
| Max gain + min VREF, at VSENSOR = −100 mV | **0.206 V** — Maintains a safe 206 mV margin above ground |

**The architecture successfully accommodates a worst-case 1 % tolerance stack-up without risking ADC clipping.** This mathematical proof confirms the resilience of the headroom strategy.

### Op-amp input offset voltage (sensitivity analysis)

Each op-amp's intrinsic input offset (`Vos`) is amplified by the subsequent gain stages:

| Op-amp | Noise gain | Downstream gain | Multiplier to VADC |
|---|---:|---:|---:|
| U1A (Stage 1) | 1.590 | 8.333 | **13.25** |
| U1B (Stage 2) | 10.00 | 0.8333 | **8.33** |
| U2A (Stage 3) | 1 + 30k/(36k∥30k) = 2.833 | 1.000 | **2.83** |
| U2B (Stage 4) | 2.000 | 1.000 | **2.00** |

If all four JFET devices exhibited maximum offset in the same polarity, the cumulative effect at the output is **26.4 × Vos**. When referred back to the input, this equates to **1.99 × Vos**. Because this static error is highly deterministic, the architecture allows it to be completely nullified via a single-point zero calibration in the microcontroller firmware.

### Future Analysis
Detailed analysis of noise, PSRR, CMRR, temperature drift, and dynamic settling time is planned for future system characterization alongside physical validation.

---

**Previous:** [03 — Signal Path](03_Signal_Path.md) · **Next:** [05 — LTspice Simulation](05_LTspice_Simulation.md)
