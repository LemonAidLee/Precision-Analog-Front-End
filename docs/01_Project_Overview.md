# 01 — Project Overview

[← Documentation index](README.md)

---

## 1. What I designed

This project presents the design and simulation of a **precision analog front end (AFE)**: the hardware that sits between a low-level analog sensor and a digital measurement system.

Concretely, I developed a complete KiCad Rev A design — encompassing a fully routed 78 mm × 100 mm two-layer PCB with a GND pour on both layers, and a comprehensive set of LTspice models — for a board that:

- accepts a **±100 mV bipolar single-ended sensor signal** on a 2-pin header,
- **band-limits** it with a 2nd-order Butterworth low-pass at 1.06 kHz,
- **amplifies** it by a factor of 10,
- **re-centres** it on a 1.65 V reference so it is unipolar,
- presents the result to a **16-bit ADS1115** ADC on channel A0,
- and connects that ADC to an **ESP32-WROOM-32** dev board over I²C.

The ±100 mV input requirement is silkscreened on the current board (`±100 mV SENSOR INPUT`) and defines the target sweep range evaluated in the LTspice DC analysis.

## 2. The engineering problem

An ADC operates as a comparator against a reference. It measures accurately only when the signal it receives is:

1. **inside** its input range,
2. **filling** a useful fraction of that range,
3. **band-limited** relative to how fast it samples,
4. **low-impedance** enough to charge the converter's sampling capacitor.

A raw sensor output generally satisfies none of these. Take the case this project targets — a sensor producing ±100 mV, fed to an ADS1115 running on 3.3 V with a ±4.096 V full-scale range:

| Property of the raw signal               | Consequence at the ADC                                                                                                                                                      |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Peak amplitude 100 mV vs. 4.096 V FSR    | Uses 2.4 % of the converter's span. The signal is quantised into ~800 of 32 768 available codes; ~5 of the 16 bits do useful work.                                          |
| Swings to −100 mV                       | Below the converter's negative supply rail. In single-ended mode this half of the waveform is simply lost, and at sufficient amplitude the input protection diodes conduct. |
| Unbounded bandwidth                      | Any interference above half the sample rate folds down into the measurement band and is indistinguishable from real signal.                                                 |
| Source impedance unknown / possibly high | The ADC's switched-capacitor input draws charge in bursts; a high-impedance source cannot settle between them, producing a gain error.                                      |

This analog front end systematically addresses all four limitations before the converter ever sees the signal. By carefully conditioning the input, the architecture ensures that the 16-bit converter delivers its full potential, turning 5 usable bits into an effective 14 bits of resolution.

## 3. What each function contributes

I divided the analog front end into dedicated functional stages, allowing each to be designed and evaluated independently before being integrated into the complete signal chain.

| Function                                   | What it fixes                                               | Implementation strategy                                                                                                                       |
| ------------------------------------------ | ----------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Filtering**                        | Out-of-band content that would alias or add broadband noise | 2nd-order Sallen-Key Butterworth low-pass, f₀ = 1.06 kHz. Positioned before any gain so interference is removed *before* it is multiplied by 10. |
| **Amplification**                    | Poor use of ADC span; quantisation noise                    | ×10 non-inverting stage, placed after the filter.                                                                                         |
| **Level shifting**                   | Bipolar signal vs. unipolar converter                       | Inverting summing amplifier that adds a 1.65 V reference and scales by 0.833, followed by a unity inverter to restore polarity.            |
| **Buffering / low output impedance** | ADC sampling-capacitor settling                             | The final op-amp stage drives the ADC directly from an op-amp output (≈ ohms), providing the necessary low-impedance source.                              |
| **Reference generation**             | Where the "centre" of the unipolar window comes from        | Resistive divider from the same 3.3 V rail that supplies the ADC, so the reference and converter track each other.                             |

Noise is addressed *architecturally* through these design choices — filtering before gain, keeping the analog and digital halves of the board physically apart, and running the analog stages from clean ±12 V rails.

## 4. Engineering verification strategy

This documentation maintains a strict distinction between design intent and verification methods to ensure credibility and traceability:

| Level                             | Meaning                                                             | Example in this project                                                      |
| --------------------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Design intent**           | What the circuit is mathematically designed to do | "f₀ = 1/(2π·1.5 kΩ·100 nF) = 1061 Hz"                                   |
| **Simulation verification** | What the SPICE model establishes                      | "LTspice, TL082 macromodel, DC sweep: VADC = 2.96220 V at VSENSOR = +100 mV" |
| **Physical validation**     | Hardware bench characterization                     | **Planned for the next development stage.**                                 |

Where a document in this repository states a number, its origin (calculated or simulated) is clearly identified.

## 5. Current status

### Current Development Stage
The current project focuses on circuit development, simulation, PCB implementation, and system architecture. 

- **System architecture**: Complete
- **Analog front-end design**: Complete
- **LTspice simulation**: Complete (DC transfer verified against the TL082 model; AC/transient verified against OP1177)
- **KiCad schematic**: Complete (Rev A)
- **PCB layout**: Complete (including component placement, full routing, and GND copper pour)
- **Power architecture**: Complete (On-board ±12 V linear regulation via LM7812/LM7912 from an external unregulated rail)
- **Digital interface**: Complete (ADS1115 and ESP32 hardware interface definition)
- **Documentation**: Complete

### Future Development
Future development will extend the validated design into physical hardware.

- **Phase 1** — PCB fabrication and assembly
- **Phase 2** — Laboratory characterization and hardware bench measurements
- **Phase 3** — Sensor integration
- **Phase 4** — ESP32 firmware development
- **Phase 5** — Input protection and EMC optimization

**Next:** [02 — System Architecture](02_System_Architecture.md)
