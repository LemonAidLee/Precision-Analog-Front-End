# 03 — Materials and Tools

This document answers: what to buy now, what to wait on, what tools are actually needed, and what can be done without professional lab equipment.

**Budget target for Section 1:** roughly **RM150–250**, excluding equipment that can be borrowed or accessed elsewhere.

---

## Section 1 — Buy Now

Generic components needed to start breadboarding the first prototype. Exact part numbers aren't critical here — these choices are unlikely to change based on later circuit calculations.

| Item                                                            | Recommended Quantity | Purpose                                     | Why We Need It                                                                                              | Priority  |
| --------------------------------------------------------------- | -------------------: | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | --------- |
| ESP32 development board                                         |                    1 | Reads the ADC and sends data to a computer  | The "brain" that connects the analog front end to the PC                                                    | Essential |
| ADS1115 ADC module                                              |                    1 | Converts analog voltage to a digital number | Lets the microcontroller read the analog signal                                                             | Essential |
| MT3608 boost converter module                                   |                  1–2 | Steps 5V up to a higher raw voltage         | Needed to generate the raw rail(s) for linear regulation                                                    | Essential |
| Positive linear regulator (e.g., 7812-type)                     |                  1–2 | Produces a clean positive rail              | Removes switching noise from the boost converter output                                                     | Essential |
| Negative linear regulator (e.g., 7912-type)                     |                  1–2 | Produces a clean negative rail              | Op-amps typically need a negative supply for bipolar signals                                                | Essential |
| Operational amplifier (general-purpose, e.g., TL074 or similar) |       1 (multi-pack) | Strengthens/buffers the sensor signal       | An electronic building block used to amplify, buffer, or modify an analog signal                            | Essential |
| 1% metal-film resistor assortment                               |                1 kit | General circuit resistors                   | Precision resistors improve measurement accuracy; a kit covers most values while exact values are still TBD | Essential |
| Ceramic capacitor assortment                                    |                1 kit | Filtering, decoupling                       | Used for noise filtering and stabilizing supply pins                                                        | Essential |
| Electrolytic capacitor assortment                               |                1 kit | Power supply smoothing                      | Needed for bulk filtering on the regulator stages                                                           | Essential |
| Solderless breadboard (e.g., 830-point)                         |                  1–2 | Prototyping platform                        | Lets you build and change the circuit without soldering                                                     | Essential |
| Dupont jumper wires (M-M, M-F, F-F)                             |                1 kit | Circuit wiring                              | Needed to connect breadboard, modules, and ESP32                                                            | Essential |
| Pin headers/connectors                                          |                1 kit | Module connections                          | Many modules need headers soldered on before breadboard use                                                 | Essential |
| Digital multimeter                                              |                    1 | Measures voltage/current/resistance         | Basic tool for checking supply rails and verifying components                                               | Essential |

---

## Section 2 — Components That Must Remain TBD

These depend on circuit calculations and simulation that haven't happened yet. Buying them now risks buying the wrong value or spec.

| Component | Why It Is TBD | What Must Be Determined |
| --- | --- | --- |
| Negative-voltage DC-DC converter (exact module/topology) | Depends on required negative rail current and noise budget | Required voltage, current, and acceptable ripple before regulation |
| Exact op-amp part number | Depends on required bandwidth, noise, and offset performance | Signal bandwidth, noise requirements, and supply voltage range |
| Exact resistor values | Depends on required gain and filter cutoff | Gain equations and filter cutoff frequency from circuit design |
| Exact capacitor values | Depends on required filter cutoff and stability | Same as above — set by filter design, not guessed |
| Input protection components (e.g., clamping diodes, series resistors) | Depends on the actual sensor/signal source characteristics | What voltage/current faults are possible at the input |
| Voltage reference (if needed for the ADC) | Depends on required measurement accuracy | Whether the ADC's internal reference is accurate enough, or an external one is needed |
| Sensor / input source | Not yet finalized | Which physical quantity will be measured first |

**Selection order:** Requirements → Circuit calculations → SPICE simulation → Component selection. Nothing in this table should be purchased before that process narrows it down.

---

## Section 3 — Tools I Need Personally

### Essential

- **Digital multimeter** — checks voltages, continuity, and resistance during build and debug.
- **Computer** — for LTspice simulation, ESP32 programming, and data logging.
- **USB cable** — connects the ESP32 to the computer.
- **Basic hand tools** — small screwdriver, wire strippers/cutters if working with headers or wires.

### Optional / Later

- Oscilloscope
- Signal generator
- Laboratory power supply
- Electronic load

> **The oscilloscope and signal generator are NOT required to begin the project.**

They become useful later for professional-grade characterization (verifying noise, ripple, and dynamic behavior), but the first working prototype does not depend on owning them. If accessible later through a university lab, makerspace, or workplace, they can be used to validate the design more rigorously — but they're not a blocker.

---

## Section 4 — What We Can Do Without an Oscilloscope

The initial prototype can be built and validated using only simulation, a multimeter, and the ADC itself:

- **LTspice/SPICE simulation** — predicts circuit behavior before building anything.
- **Multimeter measurements** — verifies DC voltages and rough current draw.
- **ADS1115 measurements** — the ADC itself can sample and report the analog signal.
- **ESP32 data logging** — reads ADC values and sends them to the computer.
- **PC plotting** — visualizes logged data as a graph.
- **Software-generated test signals** — where a physical signal source isn't available, known digital/PWM signals can substitute for basic checks.

```text
LTspice
   ↓
Predict behavior

Breadboard
   ↓
Build circuit

Multimeter
   ↓
Check DC voltages/current

ADS1115
   ↓
Measure analog signal

ESP32
   ↓
Send measurements

PC
   ↓
Plot and analyze
```

### What an oscilloscope would add later

- Seeing high-frequency ripple directly
- Seeing the actual waveform shape (not just sampled points)
- Detecting unwanted oscillation
- Observing switching noise from the DC-DC converter
- Measuring fast transients

The first prototype is still valid without one — an oscilloscope simply gives finer-grained visibility for later characterization.

---

## Section 5 — Shopping Guidance (Malaysia / Shopee)

Search terms rather than specific sellers, since sellers change over time. Prices are rough estimates only — always compare listings.

| Search term | Rough price range (RM) |
| --- | ---: |
| "ESP32 development board" | 15–35 |
| "ADS1115 16 bit ADC module" | 8–20 |
| "MT3608 boost converter" | 3–8 |
| "TL074 DIP-14" (or general-purpose quad op-amp) | 3–10 |
| "7812 7912 voltage regulator" | 3–10 |
| "1% metal film resistor assortment kit" | 10–20 |
| "ceramic capacitor assortment kit" | 8–15 |
| "electrolytic capacitor assortment kit" | 10–20 |
| "MB102 830 point breadboard" | 5–15 |
| "DuPont jumper wire kit" | 5–12 |
| "pin header connector kit" | 5–10 |
| "digital multimeter" (basic) | 15–40 |

Prices are not fixed and will vary by seller, quantity, and promotions.

---

## Section 6 — Buying Strategy

### Buy immediately

Generic, inexpensive components unlikely to change with later circuit decisions — everything in **Section 1**. These are safe purchases regardless of final gain/filter values.

### Wait until simulation

Anything in **Section 2** — values and part numbers that depend on circuit calculations and SPICE results. Buying these early risks mismatched specs and wasted money.

### Don't buy yet

Professional lab equipment (oscilloscope, signal generator, lab power supply, electronic load). These can be accessed later through a university lab, makerspace, or workplace, and are not required to get a working first prototype.

---

## Summary

1. **Buy now:** Section 1 — generic essentials, ~RM150–250.
2. **Wait until after simulation:** Section 2 — anything circuit-value-dependent.
3. **Tools actually needed:** multimeter, computer, USB cable, basic hand tools.
4. **Without an oscilloscope:** LTspice + multimeter + ADS1115 + ESP32 + PC plotting is enough to start.
5. **Equipment to access later:** oscilloscope, signal generator, lab power supply, electronic load — via lab/makerspace/workplace, not a personal purchase.
