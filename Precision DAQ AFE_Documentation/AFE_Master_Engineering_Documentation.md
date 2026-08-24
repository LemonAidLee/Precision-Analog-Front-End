# Precision Analog Front End (AFE) with Hybrid Dual-Rail Power Supply
## Complete Engineering Documentation — Stages 1–8, System-Level Failure, and Final Verified Rebuild

**Status legend used throughout this document:**
🟢 SIMULATED / MEASURED — an actual LTspice result
🔵 CALCULATED — a hand/derived calculation from measured or design values
🟡 DESIGN ASSUMPTION — chosen by the designer, not physically measured
🔴 FAILED — did not meet requirement
🟠 TBD — not yet simulated; do not purchase yet

---

## 0. Project Overview

### What is this project?

This project is a miniature precision analog front end designed to demonstrate concepts used in data acquisition and automated test equipment.

Think of the AFE as a "translator and cleaner" between a real-world sensor and an ADC.
The sensor produces a small, messy analog signal.

The AFE:

1. Provides appropriate power.
2. Amplifies the useful signal.
3. Filters unwanted noise.
4. Shifts/conditions the signal.
5. Sends a clean signal to the ADC.

This is valuable for an analog/mixed-signal hardware engineering portfolio. It is a student-scale demonstration of similar engineering concepts, rather than an exact equivalent to industrial NI hardware.

### Signal chain

```
Sensor (±100 mV, DESIGN ASSUMPTION)
   ↓
Stage 5 — 2nd-order Sallen-Key low-pass filter (~1 kHz cutoff)
   ↓
Stage 6 — Non-inverting precision amplifier (10×)
   ↓
Stage 7 — Summing/inverting level shifter (bipolar → 0–3.3 V)
   ↓
Stage 8 — Full-chain integration test
   ↓
ADS1115 ADC (0–3.3 V) → ESP32 → PC
```

All active stages use the **OP1177** precision op-amp SPICE model, powered from **±12 V** rails.

**The headline finding of this document:** Stages 5, 6, and 7 each passed their own simulations individually, but when the complete chain was tested end-to-end in **Stage 8**, the output exceeded the ADC's 0–3.3 V input window by a wide margin. This was a **system-level gain-budget error**, not a broken component. Section 10 documents the failure; Sections 11–16 document the corrected, re-verified final design.

---

## 1. Design Requirements

| Requirement | Value | Type |
|---|---|---|
| Sensor input range | ±100 mV | 🟡 Design assumption |
| ADC input range | 0 V – 3.3 V | Fixed hardware constraint (ADS1115 @ 3.3 V) |
| ADC midpoint | 1.65 V | Derived from ADC range |
| Filter target | 2nd-order low-pass, ~1 kHz cutoff, Butterworth-like | 🟡 Design target |
| Amplifier target | ~10× gain | 🟡 Design target |
| Level shifter target | Map bipolar amplifier output into 0–3.3 V with safety margin | 🟡 Design target |
| Op-amp | OP1177 (SPICE model) | 🟢 Used throughout |
| Supply | ±12 V | 🟢 Used throughout |

---

## 2. Stage 1 — Filter Requirements

### What are we trying to do?

Implement a low-pass filter before the signal reaches the ADC.

The filter is like a security gate for frequencies.

Low-frequency signals:
→ Allowed through

High-frequency noise:
→ Increasingly blocked

### Why?

The AFE receives an analog signal which may contain unwanted high-frequency noise. The ADC should not receive unnecessary high-frequency content. A low-pass filter allows useful low-frequency content through while attenuating higher-frequency signals. We selected approximately $1\,\text{kHz}$ as the filter target.

Target:

- 2nd-order low-pass filter
- Approximately $1\,\text{kHz}$ cutoff
- Butterworth-like response
- Approximately $-40\,\text{dB/decade}$ roll-off

A 2nd-order filter has an approximately $-40\,\text{dB/decade}$ roll-off because it uses two reactive components, meaning the signal's power drops more aggressively as frequencies climb past the cutoff point.

---

## 3. Stage 2 — Filter Mathematics

### What are we trying to do?

Calculate the cutoff frequency, passband gain, and Q factor based on our selected components.

### Why?

To confirm our theoretical values meet our approximately $1\,\text{kHz}$ cutoff and Butterworth-like response targets.

Component design:
$R_1 = R_2 = 1.6\,\text{k}\Omega$
$C_1 = C_2 = 100\,\text{nF}$

Target:
$f_c \approx 1\,\text{kHz}$

### Calculation

$$
f_c = \frac{1}{2\pi RC}
$$

Substitute:

$$
f_c =
\frac{1}
{2\pi(1.6\,\text{k}\Omega)(100\,\text{nF})}
$$

### Result

$$
\boxed{f_c \approx 994.7\,\text{Hz}}
$$

### Conclusion

This is acceptable for a $1\,\text{kHz}$ target since it is very close to the goal.

---

### GAIN CALCULATION

$R_f = 5.9\,\text{k}\Omega$
$R_g = 10\,\text{k}\Omega$

### Calculation

$$
K = 1 + \frac{R_f}{R_g}
$$

Calculate:

$$
K = 1 + \frac{5.9\,\text{k}\Omega}{10\,\text{k}\Omega}
$$

$$
K = 1.59
$$

Convert to dB:

$$
\text{Gain}_{\text{dB}} = 20\log_{10}(K)
$$

### Result

$$
\boxed{\text{Gain} \approx 4.03\,\text{dB}}
$$

---

### Q CALCULATION

### Calculation

$$
Q = \frac{1}{3-K}
$$

Calculate:

$$
Q = \frac{1}{3-1.59}
$$

### Result

$$
\boxed{Q \approx 0.709}
$$

### Conclusion

Compare against Butterworth:

$$
Q_{\text{Butterworth}} \approx 0.707
$$

This means the design is very close to a Butterworth response.

---

### IMPORTANT TERMINOLOGY

- **Natural/corner frequency:** The point where the filter starts to heavily attenuate the signal.
- **$-3\,\text{dB}$ frequency:** The point where the signal power is halved. For Butterworth, this is identical to the corner frequency.
- **Passband gain:** The amplification given to the frequencies we want to keep.
- **Q:** The quality factor, indicating the damping. A $\approx 0.707$ value means maximum flatness without peaking.
- **Roll-off:** How steeply the filter blocks frequencies beyond the cutoff.

---

## 4. Stage 3 — Component Selection

| Component |                             Value | Purpose                    |
| --------- | --------------------------------: | -------------------------- |
| R1        |           $1.6\,\text{k}\Omega$ | Filter resistor            |
| R2        |           $1.6\,\text{k}\Omega$ | Filter resistor            |
| C1        |                $100\,\text{nF}$ | Filter capacitor to ground |
| C2        |                $100\,\text{nF}$ | Feedback/filter capacitor  |
| Rf        |           $5.9\,\text{k}\Omega$ | Sets op-amp gain           |
| Rg        |            $10\,\text{k}\Omega$ | Sets op-amp gain           |
| Op-amp    |                            OP1177 | Precision amplifier        |
| Supply    | $+12\,\text{V} / -12\,\text{V}$ | Op-amp supply              |

$1\%$ resistors were selected because their precision ensures our calculated cutoff frequency and passband gain stay accurate. Capacitor accuracy matters because variation in actual capacitance shifts the filter's cutoff and Q values.

---

## 5. Stage 4 — Sallen-Key Circuit Design

### What are we trying to do?

Implement the theoretical design in a Sallen-Key active filter topology.

### Why?

This topology allows us to create a 2nd-order active low-pass filter using a single op-amp.

The correct capacitor connections are:

C1:
Node B → GND

C2:
Node A → $V_{\text{out}}$

Circuit structure:

$V_{\text{in}}$
 ↓
R1
 ↓
Node A
 ↓
R2
 ↓
Node B → OP1177 non-inverting input
 ↓
C1
 ↓
GND

And:
Node A → C2 → $V_{\text{out}}$

Op-amp gain network:
$V_{\text{out}}$ → Rf → inverting input
inverting input → Rg → GND

Power:
$+12\,\text{V}$
$-12\,\text{V}$

Each section contributes to the whole: R1/R2 and C1/C2 set the filter frequency; Rf and Rg set the gain and Q factor; and the OP1177 actively buffers the signal and prevents loading.

![[Pasted image 20260820120740.png]]

> **Figure:** The Sallen-Key low-pass filter topology layout.

---

## 6. Stage 5 — Low-Pass Filter Simulation & Validation

This section contains the complete simulation history. It demonstrates the engineering validation process.

### 6.1 AC Analysis

### What are we trying to do?

Perform an AC frequency sweep. Simulation command:

```text
.ac dec 100 10Hz 100kHz
```

### Why?

AC analysis tells us how gain changes with frequency.

### Verification

Record:
Passband:
$\approx +4.03\,\text{dB}$

$-3\,\text{dB}$ frequency:
$\approx 997.38\,\text{Hz}$

Theoretical target:
$\approx 998\,\text{Hz}$

### Calculation

Calculate percentage error relative to $1\,\text{kHz}$:

$$
\text{Error} =
\frac{|1000-997.38|}
{1000} \times 100\%
$$

$$
\boxed{\text{Error} \approx 0.26\%}
$$

### Conclusion

PASS

![[AFE_SallenKey_OP1177_BodePlotAnalysis.png]]

> **Figure 6.1:** LTspice AC analysis showing the approximately $997\,\text{Hz}$ $-3\,\text{dB}$ frequency.

### 6.2 Ideal E-Source Verification

### What are we trying to do?

Simulate the circuit using an ideal amplifier model (E-source) instead of a real OP1177.

### Why?

The goal was to separate:
"Does the filter topology work?"
from:
"Does the real op-amp behave ideally?"

The ideal amplifier allows us to verify the filter independently.

### Verification

Measured:
At approximately $10.028\,\text{kHz}$:
$-36.1126\,\text{dB}$

At $100\,\text{kHz}$:
$-76.0604\,\text{dB}$

### Calculation

$$
\text{Roll-off} \approx -39.95\,\text{dB/decade}
$$

Theoretical:

$$
-40\,\text{dB/decade}
$$

### Conclusion

PASS

This confirms the expected 2nd-order low-pass behavior.

![[BodePlot.png]]

> **Figure 6.2:** LTspice AC analysis with the ideal E-source verifying the $-40\,\text{dB/decade}$ roll-off.

### 6.3 Real OP1177 Verification

### What are we trying to do?

Replace the ideal amplifier with the real OP1177 SPICE model.

### Why?

To verify how the real-world component impacts the filter response.

### Verification

Results:
Passband:
$\approx +4.03\,\text{dB}$

$-3\,\text{dB}$ frequency:
$\approx 997\,\text{Hz}$

### Conclusion

PASS for the intended $\approx 1\,\text{kHz}$ operating region.

At approximately $10\,\text{kHz}$:
$\approx -36.21\,\text{dB}$

At $100\,\text{kHz}$:
$\approx -38.15\,\text{dB}$

**IMPORTANT:**
The real OP1177 has finite frequency response. At sufficiently high frequencies, the amplifier no longer behaves like the ideal amplifier assumed by the basic filter equations. Therefore the high-frequency response deviates from the ideal $-40\,\text{dB/decade}$ response. This is a realistic limitation of the real op-amp model. Do NOT calculate this as the filter's roll-off and call it $-1.94\,\text{dB/decade}$.

![[AFE_SallenKey_OP1177_BodePlotAnalysis.png]]

> **Figure 6.3:** AC analysis using the real OP1177 showing realistic high-frequency deviations.

### 6.4 Transient Analysis

AC analysis tells us how gain changes with frequency.
Transient analysis shows what happens to an actual waveform over time.
Three tests were performed.

#### 6.4.1 100 Hz Test

### What are we trying to do?

Simulate a low-frequency input signal.

Input:
`SINE(0 1 100)`

Simulation:
`.tran 0 50m 0 10u`

### Verification

Observed:
Input $\approx \pm 1\,\text{V}$
Output $\approx \pm 1.59\,\text{V}$
Gain $\approx 1.59\times$

### Conclusion

PASS

$100\,\text{Hz}$ is well below the $\approx 1\,\text{kHz}$ cutoff, so it should pass through with approximately the full passband gain.

![[Transient_100Hz.png]]

> **Figure 6.4:** Transient analysis at $100\,\text{Hz}$ showing full passband gain.

#### 6.4.2 1 kHz Test

### What are we trying to do?

Simulate an input signal at the cutoff frequency.

Input:
`SINE(0 1 1000)`

Simulation:
`.tran 0 10m 0 1u`

### Calculation

Expected:

$$
V_{\text{out,peak}} \approx
\frac{1.59}{\sqrt{2}}
$$

$$
\boxed{V_{\text{out,peak}} \approx 1.125\,\text{V}}
$$

### Verification

Observed:
Approximately $1.12\text{--}1.15\,\text{V}$ peak.

### Conclusion

PASS

The signal is approximately at the filter's cutoff, so the output should be approximately $3\,\text{dB}$ below the passband.

![[Transient_1kHz.png]]

> **Figure 6.5:** Transient analysis at $1\,\text{kHz}$ showing $3\,\text{dB}$ attenuation.

#### 6.4.3 10 kHz Test

### What are we trying to do?

Simulate a high-frequency noise signal.

Input:
`SINE(0 1 10000)`

Simulation:
`.tran 0 2m 0 100n`

### Verification

Observed:
Large startup transient followed by a very small steady-state output.

### Conclusion

PASS

The capacitors initially have stored voltage conditions determined by the simulation startup, and the circuit takes some time to settle. This initial transient should NOT be confused with the steady-state filter response. The steady-state output is strongly attenuated.

![[Transient_10kHz.png]]

> **Figure 6.6:** Transient analysis at $10\,\text{kHz}$ showing strong steady-state attenuation.

### 6.5 Cross-check against actual full-chain (Stage 8A) measured data

The passband-to-cutoff and high-frequency attenuation figures above were re-confirmed later using the actual measured full-chain values captured during Stage 8A (Section 8):

At the passband (100 Hz): $V_{filter}\approx158.99\,\text{mV}$ from a 100 mV sensor input.
At 1 kHz (near cutoff): $V_{filter}\approx112\,\text{mV}$ (both 🟢 measured).

$$
20\log_{10}\!\left(\frac{112}{158.99}\right) \approx \boxed{-3.04\,\text{dB}}
$$

This is within measurement/component tolerance of the theoretical Butterworth cutoff definition:

$$
\frac{V}{V_{passband}} = \frac{1}{\sqrt2} \approx 0.707 \;\Rightarrow\; -3.01\,\text{dB}
$$

At 10 kHz: $V_{filter}\approx1.536\,\text{mV}$ from a 100 mV sensor input.

$$
20\log_{10}\!\left(\frac{1.536\,\text{mV}}{100\,\text{mV}}\right)\approx-36.27\,\text{dB relative to sensor input}
$$
$$
20\log_{10}\!\left(\frac{1.536\,\text{mV}}{158.99\,\text{mV}}\right)\approx-40.30\,\text{dB relative to passband}
$$

Both figures are consistent ($-40.30+4.03\approx-36.27\,\text{dB}$), confirming strong high-frequency noise rejection. **PASS.**

**Stage 5 conclusion: COMPLETE, PASS. Not changed in the final redesign (see Section 11).**

---

## 7. Stage 6 — Precision Amplifier Design and Simulation

### 7.1 Objective

The objective of Stage 6 is to design and simulate the precision amplifier stage of the Analog Front End (AFE). The amplifier's job is to take the representative small sensor signal and make it larger while preserving the waveform.

The amplifier is like a "volume knob" for the sensor signal. Our representative sensor produces a small signal, and the amplifier increases the signal so that later stages can make better use of the available ADC range. This is a design assumption for the prototype, not a measurement from a physical sensor.

### 7.2 Design Requirement

The chosen representative sensor input is:

$$
V_{in}=\pm100\,\text{mV}
$$

This is a representative sensor signal. It is a DESIGN ASSUMPTION and has NOT been physically measured. It was chosen to give the project a meaningful precision-amplification problem.

The target amplifier output is:

$$
V_{out}=\pm1\,\text{V}
$$

Therefore the required gain is:

$$
A_v = \frac{V_{out}}{V_{in}}
$$

$$
A_v = \frac{1\,\text{V}}{0.1\,\text{V}}
$$

$$
\boxed{A_v=10}
$$

A $10\times$ amplifier is useful for this project because it scales the assumed $\pm100\,\text{mV}$ sensor signal up to $\pm1\,\text{V}$, making it much easier to process, filter, and measure with standard ADCs while still allowing plenty of headroom before reaching the power supply limits.

### 7.3 Choosing the Representative Sensor Input

The $\pm100\,\text{mV}$ signal was selected as a reasonable engineering design assumption for demonstrating small-signal analog conditioning. It provides a realistic challenge for the amplifier without being so small that noise completely dominates the design at this stage.

Clearly distinguish:
DESIGN ASSUMPTION: $\pm100\,\text{mV}$ sensor signal
PHYSICAL MEASUREMENT: Not yet performed.

### 7.4 Amplifier Gain Calculation

The amplifier uses a non-inverting op-amp configuration. The non-inverting configuration was selected because it presents a very high input impedance to the sensor, meaning it won't draw significant current from the delicate sensor signal.

Use:

$$
A_v=1+\frac{R_F}{R_G}
$$

The selected values are:

$$
R_G=10\,\text{k}\Omega
$$

$$
R_F=90\,\text{k}\Omega
$$

Calculate:

$$
A_v=1+\frac{90\,\text{k}\Omega}{10\,\text{k}\Omega}
$$

$$
\boxed{A_v=10}
$$

### 7.5 Amplifier Circuit Design

The circuit topology is a non-inverting amplifier.

The non-inverting input receives the sensor signal. The inverting input receives the feedback network.

The feedback network consists of:
- $R_F = 90\,\text{k}\Omega$
- $R_G = 10\,\text{k}\Omega$

The OP1177 is powered from:

$$
V_+=+12\,\text{V}
$$

$$
V_-=-12\,\text{V}
$$

Negative feedback is like a thermostat for the amplifier. It takes a fraction of the output signal and feeds it back to the inverting input, allowing the op-amp to continuously adjust and correct its output so that it perfectly matches the desired gain. This keeps the amplifier stable and accurate.

![[AFE_Amplifier_Topology.png]]

> **Figure 7.1:** Final Stage 6 non-inverting OP1177 amplifier schematic used for LTspice validation.

### 7.6 Component Selection

| Component | Value | Purpose |
|---|---:|---|
| OP1177 | OP1177 | Precision operational amplifier |
| Rf | $90\,\text{k}\Omega$ | Sets amplifier gain |
| Rg | $10\,\text{k}\Omega$ | Sets amplifier gain |
| Supply | $+12\,\text{V} / -12\,\text{V}$ | Powers the op-amp |

### 7.7 LTspice Simulation Setup

The amplifier was tested independently before being combined with the rest of the AFE. This approach allows us to isolate amplifier problems before integrating the complete signal chain, making debugging much easier.

The SPICE model used is the OP1177. The simulation is powered by $\pm12\,\text{V}$ supplies.

The following simulation types were performed:
- Operating point
- Transient analysis
- AC analysis

### 7.8 Test 1 — Positive DC Gain

Input:

$$
V_{in}=+0.1\,\text{V}
$$

Expected:

$$
V_{out}=+1\,\text{V}
$$

Actual LTspice result:

$$
V_{out}=0.999638\,\text{V}
$$

Calculate the measured gain:

$$
A_v=\frac{0.999638}{0.1}
$$

$$
\boxed{A_v=9.99638}
$$

Calculate gain error:

$$
\text{Error}=\frac{|10-9.99638|}{10}\times100\%
$$

$$
\boxed{\text{Error}\approx0.036\%}
$$

Also document the feedback-node result:

$$
V_+=0.100000\,\text{V}
$$

$$
V_-\approx0.099982\,\text{V}
$$

The nearly equal input voltages demonstrate that the negative feedback is functioning correctly, keeping the inverting and non-inverting inputs virtually shorted.

Result: PASS

### 7.9 Test 2 — Negative DC Gain

Input:

$$
V_{in}=-0.1\,\text{V}
$$

Expected:

$$
V_{out}=-1\,\text{V}
$$

Actual:

$$
V_{out}=-1.00001\,\text{V}
$$

Measured gain:

$$
A_v=\frac{-1.00001}{-0.1}
$$

$$
\boxed{A_v\approx10.0001}
$$

Feedback node:

$$
V_+\approx-0.100000\,\text{V}
$$

$$
V_-\approx-0.0999826\,\text{V}
$$

This confirms correct amplification for both positive and negative sensor signals.

Result: PASS

### 7.10 Test 3 — 100 Hz Transient

Input: SINE(0 0.1 100)
Simulation: .tran 0 50m 0 10u

This test verifies the time-domain behavior of the amplifier with a low-frequency AC signal, checking for proper gain, polarity, and waveform integrity.

Expected:

$$
V_{in,peak}=100\,\text{mV}
$$

$$
V_{out,peak}\approx1\,\text{V}
$$

Actual screenshot measurement:
Approximately:

$$
V_{in,peak}\approx99.93\,\text{mV}
$$

The waveform is clean and the amplifier produces approximately $10\times$ gain.

Check:
- Correct amplitude
- Correct polarity
- Clean sine wave
- No visible clipping
- Stable waveform

Result: PASS

![[Amplifier_100Hz.png]]

> **Figure 7.2:** Transient analysis at 100 Hz showing a clean output waveform with approximately $10\times$ gain.

### 7.11 Test 4 — 1 kHz Transient

Input: SINE(0 0.1 1000)
Simulation: .tran 0 10m 0 1u

Expected:

$$
V_{in,peak}=100\,\text{mV}
$$

$$
V_{out,peak}\approx1\,\text{V}
$$

$1\,\text{kHz}$ is around the useful frequency region of the AFE, so this confirms that the amplifier itself can handle the signal without obvious distortion before the filter stage.

Result: PASS

![[Amplifier_1kHz.png]]

> **Figure 7.3:** Transient analysis at 1 kHz confirming proper amplification without obvious distortion.

### 7.12 Test 5 — AC Gain and Bandwidth

Simulation: .ac dec 100 1 10Meg

AC analysis determines how the amplifier's gain changes with frequency.

Low-frequency gain:

$$
A_v\approx10
$$

Convert to dB:

$$
A_{dB}=20\log_{10}(10)
$$

$$
\boxed{A_{dB}\approx20\,\text{dB}}
$$

The measured $-3\,\text{dB}$ point was:

$$
\boxed{f_{-3dB}=149.43921\,\text{kHz}}
$$

At the cursor:

$$
|V_{out}|=17.000323\,\text{dB}
$$

$17\,\text{dB}$ represents approximately $3\,\text{dB}$ below the $20\,\text{dB}$ passband gain, which is the standard definition of the cutoff frequency where signal power is halved.

Calculate the approximate effective gain-bandwidth product from the simulated closed-loop result:

$$
GBW\approx A_v\times BW
$$

$$
GBW\approx10(149.43921\,\text{kHz})
$$

$$
\boxed{GBW\approx1.49\,\text{MHz}}
$$

**IMPORTANT:** This is an estimate derived from the simulation result, NOT a replacement for the manufacturer's datasheet specification.

The amplifier bandwidth is much greater than the approximately $1\,\text{kHz}$ filter region.

Calculate:

$$
\frac{149.43921\,\text{kHz}}{1\,\text{kHz}}\approx149.4
$$

Therefore the amplifier bandwidth is approximately $149\times$ higher than the $1\,\text{kHz}$ filter region.

Result: PASS

![[Amplifie_AcGain&Bandwidth.png]]

> **Figure 7.4:** AC analysis demonstrating approximately 20 dB passband gain and a 149.4 kHz bandwidth.

### 7.13 Test 6 — Maximum Input / Clipping

This was a deliberate stress test.

Input:

$$
V_{in}=\pm1\,\text{V}
$$

At $10\times$ gain, ideal output:

$$
V_{out}=\pm10\,\text{V}
$$

The amplifier supply is:

$$
\pm12\,\text{V}
$$

The LTspice waveform reached approximately:

$$
V_{out}\approx\pm10\,\text{V}
$$

This is NOT the normal operating signal. It was deliberately used to investigate output headroom and clipping. The waveform remained approximately sinusoidal without obvious flattening/clipping.

Approximate supply-to-output headroom:

$$
12\,\text{V}-10\,\text{V}=2\,\text{V}
$$

Therefore:

$$
\boxed{\text{Approximate headroom}\approx2\,\text{V}}
$$

Result: PASS

**IMPORTANT:** Do not claim that this proves the physical OP1177 hardware has exactly $2\,\text{V}$ headroom. This is a SPICE-model simulation result.

### 7.14 Test 7A — Slew-Rate / 10 kHz Transient

Input: SINE(0 0.1 10000)
Simulation: .tran 0 2m 0 100n

$10\,\text{kHz}$ was chosen to push the amplifier with a fast-changing signal. This test checks whether the amplifier can follow a relatively fast signal under the normal $\pm100\,\text{mV}$ input condition.

Expected output:

$$
V_{out}\approx\pm1\,\text{V}
$$

Calculate the required slew rate:

$$
SR_{required}=2\pi f V_{peak}
$$

$$
SR_{required}=2\pi(10\,\text{kHz})(1\,\text{V})
$$

$$
\boxed{SR_{required}\approx0.0628\,\text{V}/\mu\text{s}}
$$

The simulation waveform remained clean at $10\,\text{kHz}$ under the $\pm100\,\text{mV}$ input condition, indicating no obvious slew-rate-induced distortion in this test.

Observed:
- Approximately $\pm1\,\text{V}$ output
- Clean waveform
- No visible clipping
- No obvious distortion
- Stable operation

Result: PASS

### 7.15 Stage 6 Results

| Test | Purpose | Result |
|---|---|---|
| Test 1 | $+100\,\text{mV}$ DC → $+1\,\text{V}$ | PASS |
| Test 2 | $-100\,\text{mV}$ DC → $-1\,\text{V}$ | PASS |
| Test 3 | $100\,\text{Hz}$ transient | PASS |
| Test 4 | $1\,\text{kHz}$ transient | PASS |
| Test 5 | AC gain & bandwidth | PASS |
| Test 6 | $\pm1\,\text{V}$ stress / clipping | PASS |
| Test 7A | $\pm100\,\text{mV}$ @ $10\,\text{kHz}$ | PASS |

Final Stage 6 design values:

$$
A_v=10
$$

$$
R_F=90\,\text{k}\Omega
$$

$$
R_G=10\,\text{k}\Omega
$$

$$
V_{supply}=\pm12\,\text{V}
$$

$$
V_{sensor}=\pm100\,\text{mV}
$$

$$
V_{amp}\approx\pm1\,\text{V}
$$

$$
BW\approx149.44\,\text{kHz}
$$

Cross-check against the actual measured full-chain (Stage 8B) values, where $V_{filter}=158.993\,\text{mV}$ and $V_{amp}=1.58946\,\text{V}$:

$$
A_v=\frac{V_{amp}}{V_{filter}}=\frac{1.58946}{0.158993}\approx\boxed{9.997}\quad(\text{target }10,\ \text{error}\approx0.03\%)
$$

### 7.16 Engineering Interpretation

**1. Gain** — The amplifier increases the $\pm100\,\text{mV}$ sensor signal to approximately $\pm1\,\text{V}$.

**2. Negative feedback** — The feedback network determines the gain and keeps the op-amp inputs nearly equal during normal operation.

**3. Gain-bandwidth** — The amplifier has much more bandwidth than the approximately $1\,\text{kHz}$ filter region.

**4. Headroom** — The normal $\pm1\,\text{V}$ output is far away from the $\pm12\,\text{V}$ supply rails.

**5. Stress testing** — The $\pm1\,\text{V}$ input test deliberately pushed the amplifier toward its output limits.

**6. Slew-rate behavior** — The $10\,\text{kHz}$ test did not show obvious slew-rate-induced distortion under the normal $\pm100\,\text{mV}$ input condition.

### 7.17 Stage 6 Conclusion

Stage 6 is COMPLETE.

The $10\times$ OP1177 amplifier successfully passed all planned simulation tests.

The amplifier converts the representative $\pm100\,\text{mV}$ sensor signal into approximately $\pm1\,\text{V}$.

The simulated closed-loop bandwidth is approximately $149.44\,\text{kHz}$, providing sufficient bandwidth for the approximately $1\,\text{kHz}$ signal-conditioning region used by the AFE.

PHYSICAL HARDWARE VALIDATION HAS NOT YET BEEN PERFORMED — Stage 6 is simulation-validated only.

**Stage 6 conclusion: COMPLETE, PASS. Not changed in the final redesign (see Section 11).**

---

## 8. Stage 7 — Level Shifter Design and Simulation (ORIGINAL, AS SIMULATED)

**This section documents the ORIGINAL design exactly as built and simulated — it is intentionally NOT corrected here.** The correction is in Section 11.

### 8.1 Topology (from `AFE_LevelShifter.asc` / `AFE_CompleteIntegratedCircuit.png`)

Two cascaded inverting OP1177 stages:

**Stage A — summing inverter** (U3): sums the amplifier signal and a 1.65 V DC reference through the same virtual-ground inverting node.
**Stage B — unity inverter** (U4): un-inverts Stage A's output back to the correct polarity.

| Component | Original value | Role |
|---|---:|---|
| Rin1 | $20\,\text{k}\Omega$ | Signal input resistor (summing node) |
| Rref1 | $30\,\text{k}\Omega$ | Reference input resistor (summing node) |
| Rf3 | $30\,\text{k}\Omega$ | Feedback for U3 |
| VREF1 | DC 1.65 V (ideal source) | Reference — 🟠 **never itself simulated as a real circuit** |
| Rin2 | $10\,\text{k}\Omega$ | Input resistor, U4 |
| Rf4 | $10\,\text{k}\Omega$ | Feedback, U4 |
| Op-amps | OP1177 ×2 (U3, U4) | |
| Supply | ±12 V | |

### 8.2 Original transfer function (derived from the actual resistor values)

$$
V_x = -\frac{R_{f3}}{R_{in1}}V_{AMP} - \frac{R_{f3}}{R_{ref1}}V_{REF} = -1.5\,V_{AMP}-1.65
$$
$$
V_{ADC} = -\frac{R_{f4}}{R_{in2}}V_x = 1.5\,V_{AMP}+1.65
$$

This matches the LTspice results exactly (Section 9 verifies this against measured data).

![[AFE_LevelShifter_Circuit.png]]

> **Figure 8.1** — Original Stage 7 level-shifter schematic exactly as simulated (Rin1 = 20 kΩ).

### 8.3 Stage 7 standalone test (as simulated in isolation)

A 10 kHz test signal was applied directly to the level shifter's signal input (independent of the filter/amplifier) to verify the summing/inverting topology and the 1.65 V offset in isolation. The circuit correctly produced $V_{ADC}=1.5V_{in}+1.65$ for this standalone test. **Result: PASS (topology/math verified in isolation).**

**Critical caveat:** Stage 7 was only ever validated **in isolation**, using an arbitrary standalone test signal — never against the *actual* amplitude that Stage 6 delivers. This is exactly why the error was not caught until Stage 8.

### 8.4 FINAL Redesigned Level Shifter — 🟢 LTspice-VERIFIED

Following the Stage 8 system-level failure (Section 10) and the gain-budget root-cause analysis, the level shifter's signal-input resistor was corrected:

$$
R_{in1}:\ 20\,\text{k}\Omega \;\rightarrow\; \boxed{36\,\text{k}\Omega}
$$

All other level-shifter components ($R_{ref1}=30\,\text{k}\Omega$, $R_{f3}=30\,\text{k}\Omega$, $R_{in2}=10\,\text{k}\Omega$, $R_{f4}=10\,\text{k}\Omega$, $V_{REF}=1.65\,\text{V}$) are unchanged.

$$
\text{Level-shifter signal gain} = \frac{R_{f3}}{R_{in1}} = \frac{30\,\text{k}\Omega}{36\,\text{k}\Omega} \approx 0.8333\times
$$

**Final transfer function:**

$$
\boxed{V_{ADC} = 0.8333\,V_{AMP} + 1.65\,\text{V}}
$$

This is the **FINAL, LTspice-verified** Stage 7 design. It has been independently confirmed by re-running the full-chain `.op` simulation with $R_{in1}=36\,\text{k}\Omega$ — see Section 9.3 for the schematic-derived evidence and Section 11–13 for the complete redesign derivation and margin analysis.

---

## 9. Stage 8 — Full System Integration

### 9.1 Stage 8A — Filter + Amplifier chain, AC/transient sweep (level shifter NOT yet included)

🟢 Actual measured values, $V_{sensor}=\pm100\,\text{mV}$ at each frequency:

| Frequency | $V_{sensor}$ (mV pk) | $V_{filter}$ (mV pk) | $V_{amp}$ (mV pk) | Filter gain $V_{filter}/V_{sensor}$ | Total gain $V_{amp}/V_{sensor}$ | Attenuation rel. to passband | Result |
|---:|---:|---:|---:|---:|---:|---:|---|
| 100 Hz | 100 | 158.99 | 1589 | 1.590 | 15.89 | 0.00 dB (reference) | 🟢 PASS |
| 500 Hz | 100 | 154.33 | 1543 | 1.543 | 15.43 | −0.26 dB | 🟢 PASS |
| 1 kHz | 100 | 112.00 | 1120 | 1.120 | 11.20 | −3.04 dB | 🟢 PASS |
| 2 kHz | 100 | 38.16 | 381 | 0.382 | 3.81 | −12.40 dB | 🟢 PASS |
| 10 kHz | 100 | 1.536 | 15.14 | 0.0154 | 0.151 | −40.30 dB | 🟢 PASS |

Roll-off between 1 kHz and 10 kHz (one decade): $-40.30-(-3.04)\approx\boxed{-37.3\,\text{dB/decade}}$ — consistent with the expected 2nd-order $-40\,\text{dB/decade}$ roll-off (the small deviation is because the 1 kHz point sits almost exactly at the −3 dB corner rather than deep in the stopband, and because the real OP1177 has finite bandwidth, as already characterized in Stage 5).

![[Integration_1kHz.png]]

> **Figure 9.1** — Stage 8A, 1 kHz: $V_{sensor}$ (green, ±0.1 V), $V_{filter}$ (blue, ≈±0.11 V), $V_{amp}$ (red, ≈±1.12 V).

![[Integration_10kHz.png]]

> **Figure 9.2** — Stage 8A, 10 kHz: startup transient settling to a strongly attenuated steady state ($V_{amp}\approx\pm15\,\text{mV}$, red).

**Stage 8A conclusion: PASS at every tested frequency.** The filter and amplifier work correctly together across the full 100 Hz–10 kHz range.

### 9.2 Stage 8B — Full-chain DC range test (filter + amplifier + level shifter)

This is where the failure appeared with the **original** $R_{in1}=20\,\text{k}\Omega$ design. See Section 10.

### 9.3 Stage 8C — Full-Chain DC Verification of the FINAL Redesigned Circuit ($R_{in1}=36\,\text{k}\Omega$) — 🟢 LTspice-VERIFIED

After the redesign (Sections 8.4, 11), the complete chain was re-simulated in LTspice with $R_{in1}=36\,\text{k}\Omega$. The netlist actually used for this run — `AFE_IntegrationComplete.net`, generated 2026-08-21 15:05:47 from `AFE_IntegrationComplete.asc` — confirms the corrected value directly:

```text
VREF1 N010 0 DC 1.65
Rin1 SUM VAMP 36k
Rref1 SUM N010 30k
Rf3 SUM Vx 30k
Rin2 N009 Vx 10k
Rf4 N009 Vadc 10k
```

The corresponding operating-point result file (`AFE_IntegrationComplete.raw`, same timestamp) was decoded directly, giving the following **actual LTspice output** for $V_{sensor}=+100\,\text{mV}$:

| Node | 🟢 Actual LTspice value (Rin1 = 36 kΩ) |
|---|---:|
| $V_{sensor}$ | +0.100000 V |
| $V_{filter}$ | +0.158993 V |
| $V_{amp}$ | +1.589465 V |
| $V_{ADC}$ | **+2.974337 V** |

This matches the hand-derived transfer function $V_{ADC}=0.8333\,V_{AMP}+1.65\,\text{V}$ to within 0.22 mV (the tiny residual is the same kind of real-op-amp gain deviation already characterized throughout Sections 6–7, e.g. $A_v=9.997$ vs. the ideal 10). **The +100 mV extreme — the worst case for the top rail — is now directly LTspice-confirmed to land at 2.974 V, safely inside the 0–3.3 V ADC window with ≈0.326 V of margin.**

The 0 V and −100 mV points use the same confirmed transfer function together with the (unchanged, already-measured) Stage 8B filter/amplifier outputs:

| Sensor condition | $V_{filter}$ | $V_{amp}$ | $V_{ADC}$ | Source |
|---|---:|---:|---:|---|
| $V_{sensor}=-100\,\text{mV}$ | −0.158998 V | −1.58988 V | **0.3251 V** | 🔵 Calculated via confirmed transfer function |
| $V_{sensor}=0\,\text{V}$ | −2.244 µV | −0.000206 V | **1.6498 V** | 🔵 Calculated via confirmed transfer function |
| $V_{sensor}=+100\,\text{mV}$ | +0.158993 V | +1.589465 V | **2.974337 V** | 🟢 Direct LTspice extraction |

**Result: PASS.** The complete chain, with the corrected $R_{in1}=36\,\text{k}\Omega$, stays within 0.325 V–2.974 V — comfortably inside 0–3.3 V. This closes the loop on the Stage 8 failure: **original design → Stage 8 failure → root-cause (gain budget) → redesign → LTspice verification → final design**, all documented in Sections 8, 9, 10, and 11–13.

---

## 10. Stage 8 Design Error / System-Level Failure

> **The initial Stage 8 integration revealed a system-level gain-budget error. Although Stages 5, 6, and 7 individually passed their respective simulations, the combined transfer function exceeded the 0–3.3 V ADC input range.**

### 10.1 Actual measured DC results (🟢 Stage 8B, original design, Rin1 = 20 kΩ)

| Sensor input | $V_{filter}$ | $V_{amp}$ | $V_{ADC}$ | Within 0–3.3 V? |
|---|---:|---:|---:|:---:|
| 0 V | −2.24422 µV | −0.000206307 V | **1.64957 V** | ✅ |
| +100 mV | +0.158993 V | +1.58946 V | **+4.03384 V** | 🔴 **NO — 0.734 V over the top rail** |
| −100 mV | −0.158998 V | −1.58988 V | **−0.734703 V** | 🔴 **NO — 0.735 V below ground** |

**Verdict: FAIL.** The ADC would clip (or be damaged, depending on its absolute-max input rating) at both signal extremes.

### 10.2 Root-cause: the total gain budget was never checked end-to-end

Individual stage gains (using the *nominal design* values):

$$
A_{filter}\approx1.59\qquad A_{amplifier}\approx10\qquad A_{level\ shifter}=1.5
$$
$$
A_{total}=A_{filter}\times A_{amplifier}\times A_{level\ shifter}=1.59\times10\times1.5=\boxed{23.85}
$$

For $V_{sensor}=\pm100\,\text{mV}$, the AC output swing becomes:

$$
\pm100\,\text{mV}\times23.85\approx\pm2.385\,\text{V}
$$

Adding the 1.65 V offset:

$$
V_{ADC,min}\approx1.65-2.385=-0.735\,\text{V}\qquad V_{ADC,max}\approx1.65+2.385=4.035\,\text{V}
$$

This matches the measured −0.735 V / +4.034 V almost exactly (the small residual difference is real op-amp offset/gain error, already characterized in Sections 6–7).

![[AFE_CompleteIntegratedCircuit.png]]

> **Figure 10.1** — Full-chain schematic used for the Stage 8B DC sweep, showing every resistor value in the original (failing) design, including the level-shifter's $R_{in1}=20\,\text{k}\Omega$.

### 10.3 Why every stage could pass individually while the system failed

- **Stage 5** was tested with a generic 0–1 V AC source — its own passband gain (1.59×) was correctly verified, but 1.59× was never chained forward.
- **Stage 6** was tested against its *own* design assumption of a ±100 mV input directly (bypassing the filter) — it correctly hit ±1 V output, matching its 10× design target.
- **Stage 7** was tested standalone with an arbitrary test signal — the 1.5× gain and 1.65 V offset were verified as *math*, but never checked against the actual amplitude Stage 6 would deliver when fed through Stage 5 first.

No single stage was ever tested against the **actual signal amplitude it would receive from the previous stage in the real chain**. This is a classic **system-level gain-budget / ADC input-range-budget error**: each block was locally correct, but the *product* of the gains was never checked against the ADC's fixed 3.3 V window. This was not a faulty component, a wiring mistake, or an LTspice error — it was a missing system-level calculation.

This failure was corrected by redesigning Stage 7 (Section 8.4) and independently re-confirming the complete chain in LTspice (Section 9.3). The following sections walk through that redesign from first principles.

---

## 11. Final Design Recalculation — 🟢 FINAL, LTspice-VERIFIED

**Do not simply preserve the original design.** The complete chain is recalculated from first principles using the actual measured Stage 5/6 behavior, and the result has since been **independently confirmed by re-running the full-chain simulation in LTspice** (Section 9.3).

### 11.1 Choosing a safe final ADC operating window

Requirements: $V_{sensor}=\pm100\,\text{mV}$, ADC range 0–3.3 V, midpoint 1.65 V.

Rather than designing to the theoretical rail edges (0 V / 3.3 V exactly), a safety margin is reserved for:
- 1% resistor tolerance (level-shifter ratio can shift ±2%)
- OP1177 input-offset voltage (datasheet: ≤60 µV max — negligible at this signal level)
- 1.65 V reference tolerance (depends on final reference circuit — see Section 14's TBD note)
- ±12 V supply variation (negligible effect on gain/offset for a precision op-amp running well within its linear region)
- Possible sensor overrange (real sensors sometimes exceed nominal spec)
- General measurement/system noise margin

**Target chosen:** a symmetric swing of **±1.35 V about the 1.65 V midpoint**, i.e. a target ADC operating window of **≈0.30 V to ≈3.00 V** — leaving ≈0.30 V (≈9%) of headroom at each rail before any tolerance is even applied. This is deliberately more conservative than a naive ~0.2–3.1 V range.

### 11.2 Required total gain

$$
\Delta V_{target}=1.35\,\text{V} \qquad G_{total,required}=\frac{\Delta V_{target}}{V_{sensor}}=\frac{1.35\,\text{V}}{0.100\,\text{V}}=13.5
$$

Since Stage 5 and Stage 6 already satisfy their own requirements and do not need to change:

$$
A_{filter}\times A_{amplifier}=1.59\times10=15.9\quad(\text{fixed, unchanged})
$$

Required level-shifter **signal** gain:

$$
G_{LS,ideal}=\frac{G_{total,required}}{A_{filter}\times A_{amplifier}}=\frac{13.5}{15.9}\approx\boxed{0.8491}
$$

**This is the key correction: the level shifter must *attenuate* the signal slightly (gain < 1), not amplify it by 1.5×.**

### 11.3 Which stage to change

Per Section 8's transfer function, $V_{ADC}=\dfrac{R_{f3}}{R_{in1}}V_{AMP}+\dfrac{R_{f3}}{R_{ref1}}V_{REF}$. The offset term ($R_{f3}/R_{ref1}=1$, giving exactly $+1.65\,\text{V}$) is independent of the signal term ($R_{f3}/R_{in1}$) as long as $R_{ref1}$ and $R_{f3}$ stay fixed. **Only $R_{in1}$ needs to change** — Stage 5, Stage 6, $R_{ref1}$, $R_{f3}$, $R_{in2}$, and $R_{f4}$ are left untouched, per the "don't redesign what isn't broken" principle.

$$
R_{in1,ideal}=\frac{R_{f3}}{G_{LS,ideal}}=\frac{30\,\text{k}\Omega}{0.8491}\approx\boxed{35.33\,\text{k}\Omega}
$$

---

## 12. Final Component Values — 🟢 FINAL, LTspice-VERIFIED

![[AFE_FinalCompleteCircuit_AllValuesFinalized.png]]

> **Figure 12.0** — The final, fully-corrected complete signal chain: Stage 5 filter (R1=R2=1.5 kΩ, C1=C2=100 nF, Rf1=5.9 kΩ, Rg1=10 kΩ) → Stage 6 amplifier (Rg2=10 kΩ, Rf2=90 kΩ, giving the intended 10× gain) → Stage 7 level shifter (Rin1=36 kΩ — the corrected value, not the old 20 kΩ — Rref1=30 kΩ, Rf3=30 kΩ, Rin2=10 kΩ, Rf4=10 kΩ, VREF1=1.65 V ideal source). This diagram incorporates **both** corrections made after the original Stage 8 simulation: the Section 11 electrical redesign ($R_{in1}$: 20k→36k) and the Section 21 sourcing substitution ($R_1,R_2$: 1.6k→1.5k). Every value shown here is what should be physically built.

### 12.1 Redesigned resistor (Stage 7 only)

| # | Field | Value |
|---|---|---:|
| 1 | Ideal calculated value | $35.33\,\text{k}\Omega$ |
| 2 | Practical standard value (E24, 1%) | $\boxed{36\,\text{k}\Omega}$ |
| 3 | Resulting actual gain ($R_{f3}/R_{in1}$) | $30/36=0.8333$ |
| 4 | Resulting minimum ADC voltage | $0.3251\,\text{V}$ (see §13) |
| 5 | Resulting maximum ADC voltage | $2.9743\,\text{V}$ — 🟢 direct LTspice extraction (see §9.3, §13) |
| 6 | % error from ideal | $\dfrac{|36-35.33|}{35.33}\times100\%\approx\boxed{1.87\%}$ |
| 7 | Is 1% tolerance sufficient? | **Yes** — see §12.3 worst-case analysis |

A 36 kΩ, 1% metal-film resistor is a **standard E24 value**, commonly stocked in generic 1% resistor assortment kits — no exotic E96 value or custom series combination needed.

### 12.2 Stages left unchanged (already satisfy requirements)

| Stage | Component | Value | Reason unchanged |
|---|---|---:|---|
| 5 (filter) | R1, R2 | ~~1.6 kΩ~~ → **1.5 kΩ** (see §21) | fc already ≈997 Hz, PASS at design time; later changed to 1.5 kΩ for sourcing, fc→≈1064 Hz, no downstream effect |
| 5 (filter) | C1, C2 | 100 nF | — |
| 5 (filter) | Rf1 | 5.9 kΩ | K=1.59 already correct for filter's own spec |
| 5 (filter) | Rg1 | 10 kΩ | — |
| 6 (amp) | Rf2 | 90 kΩ | Av=10 already correct, BW 149 kHz >> 1 kHz |
| 6 (amp) | Rg2 | 10 kΩ | — |
| 7 (level shift) | Rref1 | 30 kΩ | Preserves exact 1.65 V offset |
| 7 (level shift) | Rf3 | 30 kΩ | Shared by signal & offset ratio — kept fixed |
| 7 (level shift) | Rin2 | 10 kΩ | Unity 2nd-stage inverter, unaffected |
| 7 (level shift) | Rf4 | 10 kΩ | — |

**Net change to the entire physical circuit: exactly one resistor** — $R_{in1}$: 20 kΩ → 36 kΩ.

### 12.3 Worst-case 1% tolerance stack-up (all resistors simultaneously at worst-case extreme — pessimistic bound)

| Term | Nominal | Worst-case range (±2% per ratio, two 1% resistors) |
|---|---:|---:|
| $A_{filter}$ | 1.59 | 1.559 – 1.622 |
| $A_{amplifier}$ | 10.0 | 9.80 – 10.20 |
| $G_{LS,signal}$ | 0.8333 | 0.8167 – 0.8500 |
| $G_{LS,offset}$ (rel. to 1.65 V) | 1.000 | 0.98 – 1.02 |
| 2nd-stage unity inverter | 1.000 | 0.98 – 1.02 |

Combining the absolute pessimistic case (every tolerance aligned in the worst simultaneous direction):

$$
V_{ADC,max,worst-case}\approx3.151\,\text{V}\quad(\text{margin }0.149\,\text{V to the 3.3 V rail})
$$
$$
V_{ADC,min,worst-case}\approx0.150\,\text{V}\quad(\text{margin }0.150\,\text{V to 0 V})
$$

Both margins remain **positive** even under this extremely conservative (all-resistors-worst-case-simultaneously) assumption, which in practice essentially never occurs — real stack-up follows a statistical (RSS) distribution and will sit much closer to the nominal 0.325 V / 2.974 V figures. **Conclusion: 1% resistor tolerance is sufficient.** 5% resistors would triple the per-ratio error and are **not** recommended — they would erode the margin to a level that is uncomfortably close to the rails once op-amp offset and reference tolerance are added.

---

## 13. Final Verification Plan (Mathematical) — 🟢 CONFIRMED BY LTSPICE

Using the **measured** Stage 8B filter/amp outputs (unchanged, since Stages 5–6 are not modified) combined with the **new, LTspice-confirmed** level-shifter gain $G_{LS}=0.8333$, offset $=1.65\,\text{V}$:

$$
V_{ADC} = 0.8333\,V_{AMP} + 1.65\,\text{V}
$$

| Sensor condition | $V_{filter}$ | $V_{amp}$ | $V_{ADC}$ (final design) | Source |
|---|---:|---:|---:|---|
| $V_{sensor}=-100\,\text{mV}$ | −0.158998 V | −1.58988 V | **0.3251 V** | 🔵 Calculated via confirmed transfer function |
| $V_{sensor}=0\,\text{V}$ | −2.244 µV | −0.000206 V | **1.6498 V** | 🔵 Calculated via confirmed transfer function |
| $V_{sensor}=+100\,\text{mV}$ | +0.158993 V | +1.589465 V | **2.9743 V** | 🟢 **Direct LTspice extraction** — `AFE_IntegrationComplete.raw`, 2026-08-21 15:05:47, $R_{in1}=36\,\text{k}\Omega$ (see §9.3) |

Verification against requirement:

$$
0\,\text{V} < 0.3251\,\text{V} = V_{ADC,min}\qquad\checkmark
$$
$$
V_{ADC,max}=2.9743\,\text{V} < 3.3\,\text{V}\qquad\checkmark
$$

Margins: **≈0.325 V (≈9.8% of full scale) at both rails**, nominal case. **≈0.15 V (≈4.5%) guaranteed even in the pessimistic worst-case tolerance stack-up (§12.3).**

Total system gain and ADC sensitivity:

$$
A_{total}=\frac{V_{ADC,max}-V_{ADC,min}}{V_{sensor,max}-V_{sensor,min}}=\frac{2.9743-0.3251}{0.200}\approx\boxed{13.25\ \text{V/V}}
$$
$$
\frac{\partial V_{ADC}}{\partial V_{sensor}}\approx13.25\,\frac{\text{mV}_{ADC}}{\text{mV}_{sensor}}
$$

**✅ Verification status:** This is no longer a hand-calculation awaiting confirmation. The +100 mV extreme (the worst case for the top rail, and the tightest margin of the two extremes once the small real-op-amp deviation is included) has been **directly extracted from the LTspice-generated `AFE_IntegrationComplete.raw` operating-point result**, produced from a netlist (`AFE_IntegrationComplete.net`) that explicitly confirms $R_{in1}=36\,\text{k}\Omega$ (see Section 9.3 for the raw netlist excerpt and extracted values). The redesign is **FINAL**.

---

## 14. Final Component Table

| Category | Component | Final Value / Model | Qty | Notes |
|---|---|---|---:|---|
| **Active** | ESP32 development board | Any standard ESP32 dev board | 1 | Reads ADS1115 over I²C |
| **Active** | ADS1115 ADC module | ADS1115, 16-bit, I²C | 1 | Run at VDD = 3.3 V; PGA = ±4.096 V (GAIN_ONE) recommended for 0–3.3 V signal |
| **Active** | Op-amp | OP1177 ×4 (as simulated) | 4 | 2× OP2177 (dual, pin/spec-compatible per ADI datasheet) is an acceptable substitute if re-verified in LTspice first |
| **Active** | Positive linear regulator | 7812-type | 1 | Generates clean +12 V rail |
| **Active** | Negative linear regulator | 7912-type | 1 | Generates clean −12 V rail |
| **Active** | Boost converter | MT3608 | 1–2 | Steps 5 V → raw rail(s) for the ±12 V linear regulators |
| **Active** | 1.65 V reference circuit | 🟠 **TBD — requires its own simulation** | — | Only ever modeled as an ideal DC source; see §16 |
| **Passive** | R — 1.6 kΩ, 1% | Filter (R1, R2) | 2 | |
| **Passive** | R — 5.9 kΩ, 1% | Filter gain (Rf1) | 1 | |
| **Passive** | R — 10 kΩ, 1% | Filter gain (Rg1), amp gain (Rg2), level-shift (Rin2, Rf4) | 4 | Buy several — same value used 4×, in 4 different roles |
| **Passive** | R — 90 kΩ, 1% | Amplifier gain (Rf2) | 1 | |
| **Passive** | R — 30 kΩ, 1% | Level-shift reference/feedback (Rref1, Rf3) | 2 | Unchanged from original design |
| **Passive** | **R — 36 kΩ, 1% (E24)** | **Level-shift signal input (Rin1) — CHANGED from 20 kΩ** | 1 | 🟢 LTspice-verified corrective fix (§8.4, §9.3, §11–12) |
| **Passive** | C — 100 nF, ceramic | Filter (C1, C2) | 2 | |
| **Passive** | C — decoupling caps for regulators | 0.33 µF (in) / 0.1 µF (out) per regulator, ceramic | 4 | Per typical 78xx/79xx datasheet recommendation |
| **Passive** | C — bulk electrolytic, regulators | 10–100 µF | 2–4 | Per typical linear-regulator application circuit |
| **Hardware** | Solderless breadboard | 830-point | 1–2 | |
| **Hardware** | Jumper wires | M-M/M-F/F-F kit | 1 kit | |
| **Hardware** | Pin headers/connectors | Assorted | 1 kit | |
| **Hardware** | Digital multimeter | Basic DMM | 1 | Primary verification tool (no oscilloscope available) |

**Removed from the original design:** nothing was removed — the level shifter's 2-op-amp topology is retained, only $R_{in1}$ changes value.

---

## 15. Final Shopping List (Optimized for Purchase — Malaysia / Shopee)

| Item | Minimum spec | Qty | Acceptable substitute | Purpose |
|---|---|---:|---|---|
| ESP32 dev board | Any WROOM-32 based board | 1 | Any ESP32 variant with I²C | MCU, sends data to PC |
| ADS1115 module | 16-bit I²C ADC, breakout board | 1 | ADS1015 (12-bit) if resolution is not critical | Digitizes $V_{ADC}$ |
| OP1177 (single) | Analog Devices OP1177, DIP/SOIC | 4 | 2× OP2177 (dual) — **re-simulate first if substituting** | All 4 op-amp stages |
| 7812 linear regulator | TO-220, +12 V out | 1 | LM7812 or equivalent | +12 V rail |
| 7912 linear regulator | TO-220, −12 V out | 1 | LM7912 or equivalent | −12 V rail |
| MT3608 boost module | Adjustable boost, ≥ raw rail voltage/current needed | 1–2 | Any adjustable boost converter module | Steps 5 V up before linear regulation |
| 1% metal-film resistor kit | E24 values incl. 1.6k, 5.9k, 10k, 30k, 36k, 90k | 1 kit | Individual 1% resistors if kit lacks 5.9k/36k/90k | All gain/filter/level-shift resistors |
| Ceramic capacitor kit | Incl. 100 nF, 0.1 µF, 0.33 µF | 1 kit | — | Filter caps + regulator decoupling |
| Electrolytic capacitor kit | 10–100 µF, ≥25 V rated | 1 kit | — | Regulator bulk filtering |
| 830-point breadboard | Standard MB102-style | 1–2 | Any full-size solderless breadboard | Prototyping |
| Dupont jumper wire kit | M-M, M-F, F-F | 1 kit | — | Wiring |
| Pin header kit | 2.54 mm, male/female | 1 kit | — | Module connections |
| Digital multimeter | DC volts, resistance | 1 | Any basic DMM | Primary hardware verification tool |

**Oscilloscope: NOT required.** All frequency-domain and transient behavior needed for this design was already verified in LTspice (Sections 6, 7, 9). Hardware bring-up only needs DC-point verification, which a multimeter handles.

**Signal generator: NOT required.** A simple adjustable resistor-divider "fake sensor" (Section 16, step 6) provides adjustable DC test points across ±100 mV, which is sufficient to verify the gain chain without AC stimulus.

---

## 16. Final Verification Plan (Hardware, Post-Purchase)

Because there is **no oscilloscope and no signal generator**, this plan is built around **multimeter + ADS1115 + ESP32**, matching what was already validated in simulation (Sections 6, 7, 9).

1. **Power supply verification** — Multimeter: confirm +12 V and −12 V rails (relative to common ground) at the regulator outputs, within regulator tolerance (e.g., 11.5–12.5 V). *Cannot verify switching ripple without a scope — accept this as an unverified residual risk, mitigated by the linear regulation stage.*
2. **1.65 V reference verification** — Multimeter (high input impedance): confirm the reference node reads 1.65 V ±1% *before* wiring it into the level shifter, to catch a bad divider/reference before it corrupts every downstream measurement.
3. **Stage 5 DC verification** — Apply a known DC test voltage (see step 6) at the sensor input; measure $V_{filter}$ at 0, +100 mV, −100 mV; expect ≈0 V, +159 mV, −159 mV (passband gain 1.59×, valid at DC).
4. **Stage 6 DC verification** — Measure $V_{amp}$ for the same three conditions; expect ≈0 V, +1.59 V, −1.59 V.
5. **Stage 7 output verification** — Measure $V_{ADC}$ for the same three conditions; expect ≈**1.65 V, 2.974 V, 0.325 V** per Section 13 (the +100 mV figure is LTspice-confirmed, not just calculated). Any large deviation points to a resistor value or reference error before the ADC is even connected.
6. **±100 mV equivalent input verification** — With no signal generator, build a simple adjustable "fake sensor": a potentiometer-based divider fed from a stable low-voltage source, adjusted and confirmed with the multimeter to sit at exactly −100 mV, 0 V, and +100 mV relative to ground. This substitutes for a real sensor for **DC/gain verification only** — it cannot verify AC/frequency response (already covered by Stage 8A simulation).
7. **ADC range verification** — With ESP32 + ADS1115 running, sweep the fake-sensor divider from −100 mV to +100 mV and confirm the ESP32-reported voltage tracks the multimeter reading and never saturates (0 or 65535 raw codes) at either extreme.
8. **ESP32/ADS1115 connection verification** — Confirm I²C address/wiring (SDA, SCL, VDD, GND, ADDR strap), and confirm a stable, low-noise reading of ≈1.65 V-equivalent code when the fake sensor is at 0 V.

**Explicitly cannot be verified without an oscilloscope:** ripple on the ±12 V rails, actual waveform shape/distortion at any frequency, and any dynamic (AC) behavior of the filter/amplifier stages on real hardware. These were already characterized in simulation (Sections 6, 7, 9) and are treated as simulation-validated only, pending future access to lab equipment.

---

## 17. Engineering Lessons Learned

### 17.1 Lessons from Stages 1–6 (Filter & Amplifier Validation)

**1. Theory must be verified by simulation** — Calculations give us the expected behavior, but simulation checks whether the actual circuit implementation matches.

**2. Don't immediately change component values when simulation is wrong** — First investigate topology, wiring, component values, model behavior, and the netlist.

**3. Use ideal models to isolate problems** — The E-source allowed us to determine whether the filter itself was correct, independent of real op-amp behavior.

**4. Real components are not ideal** — The OP1177 behaves differently from an ideal amplifier at high frequencies.

**5. The SPICE netlist is a powerful debugging tool** — An early cutoff-frequency discrepancy was eventually traced to the capacitor topology by inspecting the netlist directly.

**6. Frequency-domain and time-domain simulations complement each other** — AC analysis reveals frequency response; transient analysis reveals actual waveform behavior. Neither alone is sufficient.

### 17.2 Lessons from Stage 8 (System Integration)

**System-level gain budgeting** — Every analog stage has a gain. When stages are cascaded, **gains multiply**, not add — a 1.59× filter, a 10× amplifier, and a 1.5× level shifter don't combine to "roughly a bit more than 10×"; they combine to **23.85×**. Testing each block only against its own local design target (not the actual amplitude the previous block will hand it) hides this multiplication until the whole chain is run together. The fix is a single up-front calculation, done before any component is chosen: multiply every stage's gain, apply it to the full expected input range, and check the result against the *next* stage's valid input window — repeated all the way to the ADC.

**ADC input-range budgeting** — An ADC has a hard, fixed input window (here, 0–3.3 V). Every analog stage upstream of it must be designed **backward from that constraint**, not forward from "what gain feels reasonable." In this project, the correct design order was: *decide the safe ADC operating window → determine the total gain that fits within it → only then decide how that gain is split across stages.* The original design instead picked a level-shifter gain (1.5×) that seemed reasonable in isolation, without checking it against the amplitude the amplifier would actually deliver.

**The concrete example** — Stages 5, 6, and 7 each passed their own simulation. Stage 8 — the only place the *complete* transfer function was actually evaluated — is what caught a design error that would otherwise have gone straight to a hardware order, wasting the resistor purchase, the build time, and (worse) potentially exposing an ADC to −0.73 V, a value low enough to risk latch-up or damage on many ADC/MCU input pins.

---

## 18. Overall Project Results

| Parameter                    |                      Theoretical |                               Simulated | Status |
| ----------------------------- | --------------------------------: | --------------------------------------: | ------ |
| Filter passband gain          |      $\approx 4.03\,\text{dB}$ |             $\approx 4.03\,\text{dB}$ | PASS   |
| Filter cutoff                 |       $\approx 998\,\text{Hz}$ |           $\approx 997.38\,\text{Hz}$ | PASS   |
| Filter Q                      |                $\approx 0.709$ |                Consistent with response | PASS   |
| Ideal roll-off                |        $-40\,\text{dB/decade}$ |    $\approx -39.95\,\text{dB/decade}$ | PASS   |
| Filter $100\,\text{Hz}$ transient |           $\approx 1.59\times$ |                  $\approx 1.59\times$ | PASS   |
| Filter $1\,\text{kHz}$ transient  | $\approx 1.125\,\text{V}$ peak | $\approx 1.12\text{--}1.15\,\text{V}$ | PASS   |
| Filter $10\,\text{kHz}$ transient |               Strong attenuation |                      Strong attenuation | PASS   |
| Amplifier gain | $A_v=10$ | $A_v\approx9.996\text{--}10.0001$ | PASS |
| Amplifier bandwidth | — | $149.44\,\text{kHz}$ | PASS |
| Level shifter (isolated, original) | $V_{ADC}=1.5V_{in}+1.65$ | Matched | PASS (in isolation) |
| **Full chain, original design (Stage 8B)**, $R_{in1}=20\,\text{k}\Omega$ | $0\,\text{V}\le V_{ADC}\le3.3\,\text{V}$ | **−0.735 V to +4.034 V** | 🔴 **FAIL** |
| **Full chain, final redesign (Section 9.3, 13)**, $R_{in1}=36\,\text{k}\Omega$ | $0\,\text{V}\le V_{ADC}\le3.3\,\text{V}$ | **0.325 V to 2.974 V** | 🟢 **PASS — LTspice-verified** |

---

## 19. Interview Explanation / Project Value

**One-sentence summary:** *"I designed a 4-stage analog front end in LTspice, and system-level integration testing caught a gain-budget error that none of the individual stage tests could reveal — the corrected design changes exactly one resistor and is verified with a documented margin against the ADC's input range."*

**What this demonstrates to an interviewer:**
- Comfort with op-amp topologies (Sallen-Key active filter, non-inverting amplifier, summing/inverting level shifter) and their governing equations.
- The discipline to simulate every stage *and* the full chain — not just individual blocks.
- The ability to root-cause a system failure to a specific, quantified cause (a missing gain-budget calculation) rather than guessing or re-designing from scratch.
- A minimal, targeted fix (one resistor) instead of an over-engineered rebuild — showing respect for what already worked.
- Honest tolerance analysis (worst-case 1% resistor stack-up) rather than assuming ideal components.
- Awareness of what can and cannot be verified with the tools actually on hand (multimeter + ADC, no scope/signal generator), and a concrete plan for each.

---

## 20. Current Project Status

**Stages 1–8:** COMPLETE (simulation-validated).

**System-level redesign (Sections 8.4, 9.3, 11–13):** 🟢 **COMPLETE and LTspice-VERIFIED.** The corrected $R_{in1}=36\,\text{k}\Omega$ was re-simulated as a full-chain operating-point analysis; the resulting netlist and raw output directly confirm $V_{ADC}=2.9743\,\text{V}$ at $V_{sensor}=+100\,\text{mV}$, matching the hand-derived transfer function to within 0.22 mV. This is the **final** Stage 7/8 design — no further LTspice re-verification is required before purchasing the 36 kΩ resistor.

**1.65 V reference sub-circuit:** 🟠 TBD — modeled only as an ideal DC source in every simulation to date; needs its own design and simulation before the reference hardware is purchased. (This item is unaffected by the Rin1 redesign and remains open.)

**Hardware build/physical validation:** NOT YET PERFORMED. All results in this document are simulation-based.

### Current validated design (complete, final)

$R_1 = R_2 = 1.6\,\text{k}\Omega$
$C_1 = C_2 = 100\,\text{nF}$
$R_{f1} = 5.9\,\text{k}\Omega,\ R_{g1} = 10\,\text{k}\Omega$ (filter)
$R_{f2} = 90\,\text{k}\Omega,\ R_{g2} = 10\,\text{k}\Omega$ (amplifier)
$R_{ref1} = 30\,\text{k}\Omega,\ R_{f3} = 30\,\text{k}\Omega,\ R_{in2} = 10\,\text{k}\Omega,\ R_{f4} = 10\,\text{k}\Omega$ (level shifter, unchanged)
$\boxed{R_{in1} = 36\,\text{k}\Omega}$ (level shifter, **corrected** from 20 kΩ)
Op-amp = OP1177 ×4, Supply = $\pm12\,\text{V}$, $V_{REF}=1.65\,\text{V}$ (🟠 reference circuit TBD)

---

# FINAL BUILD — BUY THESE

*Everything below is verified against the 0–3.3 V ADC range per Section 13, including a direct LTspice re-simulation of the redesigned circuit (Section 9.3). Items marked 🟠 TBD are explicitly NOT final — do not purchase that specific item yet.*

| Item | Value / Model | Qty |
|---|---:|---:|
| ESP32 dev board | any WROOM-32 | 1 |
| ADS1115 module | 16-bit I²C, run @ 3.3 V, PGA ±4.096 V | 1 |
| OP1177 op-amp | single, DIP/SOIC | 4 |
| 7812 linear regulator | +12 V | 1 |
| 7912 linear regulator | −12 V | 1 |
| MT3608 boost converter | adjustable | 1–2 |
| Resistor 1.6 kΩ, 1% | R1, R2 | 2 |
| Resistor 5.9 kΩ, 1% | Rf1 | 1 |
| Resistor 10 kΩ, 1% | Rg1, Rg2, Rin2, Rf4 | 4 |
| Resistor 90 kΩ, 1% | Rf2 | 1 |
| Resistor 30 kΩ, 1% | Rref1, Rf3 | 2 |
| **Resistor 36 kΩ, 1%** | **Rin1 — the corrected, LTspice-verified value (was 20 kΩ)** | **1** |
| Capacitor 100 nF, ceramic | C1, C2 | 2 |
| Capacitor 0.33 µF / 0.1 µF, ceramic | regulator decoupling | 4 |
| Electrolytic capacitor 10–100 µF | regulator bulk filtering | 2–4 |
| 🟠 **1.65 V reference circuit — TBD** | requires its own simulation before purchase | — |
| Breadboard, 830-point | — | 1–2 |
| Jumper wire kit | M-M/M-F/F-F | 1 kit |
| Pin header kit | — | 1 kit |
| Digital multimeter | — | 1 |

**Not needed:** oscilloscope, signal generator (see Section 15).

**Before ordering:** the $R_{in1}=36\,\text{k}\Omega$ redesign is final and LTspice-verified (Section 9.3) — no further re-simulation is needed for that resistor. The one remaining open item is the 1.65 V reference sub-circuit (Section 14) — it was only ever an ideal source in every simulation to date and still needs its own design and simulation before that specific piece of hardware is purchased.

---

## 21. Addendum — 1.6 kΩ → 1.5 kΩ Substitution (Stage 5, R1/R2)

**Trigger:** 1.6 kΩ (and a suitable 100 Ω for a series-combo workaround) could not be sourced locally during final BOM purchasing. Verified 2026-08-21 whether a single 1.5 kΩ, 1% resistor can replace both R1 and R2 directly.

### 21.1 Where 1.6 kΩ is used and why

R1 = R2 = 1.6 kΩ are the two frequency-setting resistors of the Stage 5 Sallen-Key low-pass filter (`LTSpice_Simulation/AFE_SallenKey_OP1177_2.asc`, `AFE_IntegrationComplete.asc`), used in the **equal-component design** (R1 = R2 = R, C1 = C2 = 100 nF). For this topology:

$$f_c=\frac{1}{2\pi RC}\qquad Q=\frac{1}{3-K}$$

where $K=1+\dfrac{R_{f1}}{R_{g1}}=1+\dfrac{5.9\text{k}}{10\text{k}}=1.59$ is the passband gain set by Rf1/Rg1 — **not** by R1/R2. 1.6 kΩ with 100 nF gives $f_c\approx994.7\,\text{Hz}$ (994.7 Hz theoretical, 997.38 Hz LTspice-measured — see Section 6), meeting the ≈1 kHz Butterworth-like target from Section 1.

### 21.2 Recalculation with 1.5 kΩ

$$f_c=\frac{1}{2\pi(1.5\,\text{k}\Omega)(100\,\text{nF})}\approx\boxed{1061.03\,\text{Hz (theoretical)}}$$

Because $Q=1/(3-K)$ depends only on $K$ (Rf1/Rg1) and is **algebraically independent of R** in an equal-component Sallen-Key design, $Q\approx0.7092$ is **unchanged**, and so is the passband gain $K=1.59\times$ (4.03 dB). R1/R2 only ever appear in the $f_c$ equation.

### 21.3 Quantified deviation

$$\text{Resistor error}=\frac{|1.5-1.6|}{1.6}\times100\%=6.25\%\qquad\qquad\text{Error}(f_c)=\frac{|1061.03-994.72|}{994.72}\times100\%=6.67\%$$

(Exact, since $f_c\propto1/R$ with C fixed: $f_{c,\text{new}}/f_{c,\text{old}}=1.6/1.5=1.06\overline{6}$.)

### 21.4 Effect on downstream parameters

| Parameter | Effect of R1=R2: 1.6k→1.5k | Why |
|---|---|---|
| Filter passband/DC gain (K) | **None** — 1.59× unchanged | K set by Rf1/Rg1, untouched |
| Filter Q | **None** — 0.7092 unchanged | $Q=1/(3-K)$, R-independent for R1=R2 |
| Stage 6 gain (10×) | **None** | Different resistor pair (Rf2/Rg2 = 90k/10k), untouched |
| Stage 7 level shift / ADC range | **None** — still 0.325 V–2.975 V | Rin1(36k)/Rref1/Rf3/Rf4 untouched (Section 11.3) |
| Filter cutoff $f_c$ | **+6.67%** (≈997→≈1064 Hz) | Only parameter that moves |
| Bias-current offset | **Negligible improvement** (~3.2 µV → ~3.0 µV, OP1177 $I_b\le2\,\text{nA}$) | Lower R → slightly lower $I_b\times R$ error |
| Input loading on filter | **Negligible** (~6% lower input Z, still kΩ-range into an op-amp) | No practical loading concern either way |
| Power dissipation in R1/R2 | **Negligible** (<100 nW either value) | Signal levels are ~100 mV into kΩ |
| Overall design accuracy | **Unaffected for the DC/near-DC sensor signal** | See 21.5 |

### 21.5 Worst-case ±100 mV check and clipping/headroom

Stage 5 output is a pure gain block at DC/near-DC ($K=1.59$, unchanged): $\pm100\,\text{mV}\to\pm159\,\text{mV}$, exactly as before → Stage 6 → $\pm1.59\,\text{V}$ → Stage 7 → **0.325 V / 2.975 V at the ADC**, identical to the already-verified figures in Section 13. No clipping, no headroom loss, no change in worst-case ADC excursion. The Stage 8 physical DC verification steps in Section 16 (0, +100 mV, −100 mV) remain valid **unmodified** with 1.5 kΩ installed.

The only real consequence is the anti-aliasing corner moving from ≈997 Hz to ≈1064 Hz — irrelevant here because every verification test (Sections 6, 9, 16) drives the filter at DC/near-DC; the sensor signal of interest sits far below both cutoffs, and the roll-off is still ≈−40 dB/decade either way.

### 21.6 LTspice re-verification

Two ready-to-run schematics were prepared with R1=R2=1.5k and `.meas` directives matching the existing Section 6/16 test methodology (AC sweep for $f_c$, and a 0/+100 mV/−100 mV PWL step matching the Section 16 DC procedure): `AFE_SallenKey_R1p5k_test.asc` and `AFE_SallenKey_R1p5k_DCstep_test.asc` (baselines re-run alongside as `..._R1p6k_baseline_check.asc` / `..._R1p6k_DCstep_baseline.asc` for a same-methodology sanity check against the already-published 997.38 Hz figure). **Automated headless batch simulation (`-b`) did not complete in the assisting environment** — the process launches but never exits, reproducing even on the original unmodified file, so this is an environment limitation, not a result. The predicted $f_c\approx1063.9\,\text{Hz}$ above applies the same +0.27% real-op-amp correction already observed between the theoretical (994.7 Hz) and LTspice-measured (997.38 Hz) 1.6k baseline — a calculated extrapolation, not a fresh simulation. **Action:** open either file in LTspice (already installed) and press Run — takes seconds — to confirm the exact figure; the DC-step file's `.meas` results (`V_zero`, `V_pos100m`, `V_neg100m`) should read ≈0 V / ≈+159 mV / ≈−159 mV, matching Section 16 step 3 exactly.

### 21.7 Conclusion

**A — SAFE.** A single 1.5 kΩ, 1% resistor may directly replace both R1 and R2. Gain, ADC range, level-shifting, bias-current error, loading, and power dissipation are unaffected; the only change is the filter cutoff shifting +6.67% (≈997 Hz → ≈1064 Hz), which has no effect on the DC/near-DC sensor signal this design was verified against. The 100 Ω series-combo workaround from the shopping list is **no longer needed** — simplifies the BOM by one resistor value.

**Updated BOM line (supersedes Section 14/15):**

| Component | Old | New |
|---|---|---|
| R1, R2 (Stage 5 filter) | 1.6 kΩ, 1%, qty 2 | **1.5 kΩ, 1%, qty 2** |

No other component values change. See **Figure 12.0** for the complete final schematic with this value already applied alongside the Section 11 $R_{in1}$ correction.
