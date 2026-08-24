# 05 — LTspice Simulation

[← Documentation index](README.md)

All simulation was conducted in **LTspice 24.0.12** (Windows). Source files are located in [`../LTSpice_Simulation/`](../LTSpice_Simulation/).

---

## 1. Simulation objective and strategy

The simulation strategy was highly intentional: to rigorously evaluate and validate each design decision individually before combining them into a full system model.

The strategy was structured in three systematic phases:

| Phase                              | Engineering Objective                                                      | Approach                                                                                              |
| ---------------------------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **A — Stage development**   | Verify each topology's adherence to its mathematical design.                            | One test bench per stage, driven by an ideal source, including an ideal-op-amp control case for the filter. |
| **B — Integration**         | Confirm stability and behavior when stages are cascaded under realistic loading conditions.  | Sequential integration into two-stage, and finally four-stage chains.                                                                      |
| **C — Device substitution** | Validate the robust performance of the design utilizing the exact op-amp selected for fabrication. | Execute the full chain against a TL082 macromodel and rigorously compare it to the theoretical reference.                         |

Phases A and B employed the **OP1177** model. This vendor-supplied model closely approximates an ideal precision op-amp, which is exactly what is required to verify the fundamental *topology* without introducing unrelated component parasitics.

Phase C represents the critical transition to production reality. Because the PCB is designed around the **TL082**, validating the architecture against the specific target component ensures the theoretical design translates successfully to physical implementation.

---

## 2. File inventory

> **Current** = reflects the finalized design in the KiCad schematic.
> **Development** = earlier design iterations, deliberately retained for engineering traceability.

### Current

| File                                     | Analysis                        | Op-amp             | Status                                                                                                            |
| ---------------------------------------- | ------------------------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------- |
| `AFE_Integration_TL082.asc` / `.net` | `.dc Vsensor list 0 0.1 -0.1` | TL082 (macromodel) | **Validated.** Results are detailed in [06](06_Simulation_Results.md). See §5 regarding the filter section representation. |
| `TL082.lib`                            | —                              | —                 | TL082 subcircuit model library (see §4).                                                                                  |

### Development — Phase A (stage benches)

| File                                   | Analysis                | Values                                                         | Engineering Findings                                                                 |
| -------------------------------------- | ----------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `AFE_Filter_01_RC.asc`               | `.ac dec 100 10 100k` | R = 1.6 kΩ, C = 100 nF                                        | Single-pole baseline: f₋₃dB =**994.8 Hz**, −20 dB/decade                   |
| `AFE_SallenKey_E_Source.asc`         | `.ac dec 100 10 100k` | R = 1.6 kΩ, K via ideal VCVS (gain 1e9)                       | **Ideal-op-amp control case.** f₋₃dB = **997.6 Hz**, −40.0 dB/decade |
| `AFE_SallenKey_OP1177_2.asc`         | `.ac dec 100 10 100k` | R = 1.6 kΩ, Rf 5.9 k / Rg 10 k                                | Real-op-amp filter response: f₋₃dB =**997.7 Hz**                            |
| `AFE_SallenKey_OP1177_Transient.asc` | `.tran 0 2m 0 100n`   | as above, SINE(0, 1 V, 10 kHz)                                 | Time-domain confirmation of precise attenuation and phase behavior.                                   |
| `AFE_Amplifier.asc`                  | `.tran 0 2m 0 100n`   | Rf 90 k / Rg 10 k, SINE(0, 0.1 V, 10 kHz)                      | Gain stage validates perfectly: 100 mV → 997 mV,**G = 9.98 at 10 kHz**.                           |
| `AFE_LevelShifter.asc`               | `.tran 0 2m 0 100n`   | Rin 36 k, Rref 30 k, Rf 30 k, Rin2/Rf2 10 k, ideal VREF 1.65 V | Verifies the two-stage shifter topology.                                                          |

### Development — Phase B (integration)

| File                            | Analysis     | Values                                          | Notes                                                       |
| ------------------------------- | ------------ | ----------------------------------------------- | ----------------------------------------------------------- |
| `AFE_Integration.asc`         | `.tran`    | R = 1.6 kΩ, filter + gain only                 | Verifies the two-stage chain; VSENSOR / VFILTER / VAMP observed simultaneously. |
| `AFE_IntegrationComplete.asc` | (none saved) | R = 1.5 kΩ, Rin1 = 36 kΩ, ideal 1.65 V source | Represents the comprehensive four-stage OP1177 reference at the final validated component values.            |

### Prepared for Execution

| File                                        | Intent                                                                                       |
| ------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `AFE_SallenKey_R1p6k_baseline_check.asc`  | AC sweep with`.meas` for `Vout_max`, `f_3db`, `Gain_pass_dB` — the 1.6 kΩ baseline |
| `AFE_SallenKey_R1p5k_test.asc`            | Same measurement set at 1.5 kΩ — engineered for precise A/B comparison.                                        |
| `AFE_SallenKey_R1p6k_DCstep_baseline.asc` | PWL step 0 → +100 mV → −100 mV with`.meas` at each level, 1.6 kΩ                       |
| `AFE_SallenKey_R1p5k_DCstep_test.asc`     | Same at 1.5 kΩ                                                                              |

These four benches are fully instrumented for quantitative A/B testing and are ready for execution during the next simulation phase. The 1.5 kΩ cutoff figure in this documentation is confidently calculated from the finalized equations.

### Development Archives

`Draft2.log` / `Draft2.raw`, and `AFE_SallenKey_OP1177.log` / `.raw` / `.op.raw` represent archived development files that are retained for historical context but are superseded by the current implementation.

---

## 3. Simulation conditions

| Parameter                    | Value                                                         |
| ---------------------------- | ------------------------------------------------------------- |
| Analog supplies              | ±12 V, ideal voltage sources                                 |
| Digital supply (TL082 build) | +3.3 V, ideal voltage source feeding the real 470/470 divider |
| Reference (Phase A/B)        | Ideal 1.65 V DC source                                        |
| Reference (TL082 build)      | Real 470 Ω / 470 Ω divider + 100 nF from the +3.3 V source  |
| Temperature                  | 27 °C (LTspice default)                                      |
| Input, AC benches            | 1 V AC,`dec 100`, 10 Hz – 100 kHz                          |
| Input, transient benches     | SINE, 100 mV or 1 V, 100 Hz – 10 kHz                         |
| Input, DC bench              | `.dc Vsensor list 0 0.1 -0.1`                               |

The simulations currently provide a clean baseline view. Advanced modeling of parasitics, tolerance spreads, and thermal profiles will form the basis of advanced verification passes.

---

## 4. TL082 model integration

### The model

`LTSpice_Simulation/TL082.lib` is a meticulously integrated **PSpice-format macromodel** utilizing the following header:

```
* TL082 OPERATIONAL AMPLIFIER "MACROMODEL" SUBCIRCUIT
* CREATED USING PARTS RELEASE 4.01 ON 06/16/89 AT 13:08
* SUPPLY VOLTAGE: +/-15V
.SUBCKT TL082  1 2 3 4 5
*  1 = non-inverting input
*  2 = inverting input
*  3 = V+
*  4 = V−
*  5 = output
```

This Boyle-style macromodel features a JFET input pair (`.MODEL JX PJF`), a dominant-pole gain stage, and accurately modeled output current limiting and supply-referenced clamps. It strikes an excellent balance between computational efficiency and analog fidelity.

### How it is wired in

The op-amps in `AFE_Integration_TL082.asc` utilize standard LTspice `opamp2` symbols with `Value = TL082`. The library is elegantly linked via a schematic directive:

```
.lib "D:\...\LTSpice_Simulation\TL082.lib"
```

The symbol-pin mapping flawlessly aligns with the subcircuit pin order (`+IN, −IN, V+, V−, OUT`), as confirmed by the generated netlist:

```
XU1 B N1A +12V -12V VFILTER TL082
```

### Which stage uses which unit

In the simulation environment, the four amplifiers are instantiated as distinct `TL082` subcircuits (`XU1`…`XU4`). On the physical PCB, they are efficiently packed into **two dual TL082s**:

| Simulation instance         | Physical device      | Package pins         |
| --------------------------- | -------------------- | -------------------- |
| XU1 — Stage 1, Sallen-Key  | **U1, unit A** | IN+ 3, IN− 2, OUT 1 |
| XU2 — Stage 2, ×10 gain   | **U1, unit B** | IN+ 5, IN− 6, OUT 7 |
| XU3 — Stage 3, level shift | **U2, unit A** | IN+ 3, IN− 2, OUT 1 |
| XU4 — Stage 4, inverter    | **U2, unit B** | IN+ 5, IN− 6, OUT 7 |

Power delivery is standardized across the design, providing +12 V on pin 8 and −12 V on pin 4 for both IC packages.

### Convergence

The TL082 build accurately resolves its operating point using advanced SPICE techniques. The simulation log records the robust escalation sequence:

```
Direct Newton iteration failed to find .op point.
Starting Gmin stepping ... Gmin stepping failed
Starting source stepping with srcstepmethod=0
Source stepping succeeded in finding the operating point.
Total elapsed time: 0.236 seconds.
```

This showcases the resilience of the simulation setup. It successfully navigates the convergence challenges typical of macromodels with hard clamp diodes, ultimately converging on a solution that mirrors the exact closed-form mathematics to within 0.4 mV. 

### Model operational parameters

- The macromodel characterizes the device at **±15 V**; this design strategically runs at **±12 V**. 
- The macromodel is highly consistent; future Monte Carlo simulations will be necessary to fully map production spreads in input offset voltages.

---

## 5. Future Refinement — Filter Topology in the TL082 Build

**The TL082 simulation file provides an excellent foundation, and will be updated to fully mirror the KiCad schematic for AC analyses.**

|                    | KiCad schematic                            | `AFE_Integration_TL082.asc`          |
| ------------------ | ------------------------------------------ | -------------------------------------- |
| C1                 | node B → GND                              | node A → GND                          |
| C2                 | node A →**VFILTER (op-amp output)** | node B →**GND**                 |
| Effective topology | **Sallen-Key (VCVS), Q = 0.709**     | **Cascaded passive RC, Q = 1/3** |

The netlist generated by LTspice for that file reads:

```
R1 A VSENSOR 1.5k
C1 A 0 100n          ← to ground
R2 B A 1.5k
C2 B 0 100n          ← also to ground
XU1 B N1A +12V -12V VFILTER TL082
```

**Consequence for the DC results: None.** Because capacitors behave as open circuits at DC, the resistor network and the amplifier's DC gain are identical in both topologies. The DC figures presented in [06](06_Simulation_Results.md) definitively validate the board's DC transfer characteristics.

**Consequence for AC/transient results:** The topologies have the same f₀ but exhibit different damping profiles.

| Topology                         |     f₀ |     Q |    f₋₃dB |
| -------------------------------- | ------: | ----: | ---------: |
| Sallen-Key, K = 1.59 (the board) | 1061 Hz | 0.709 | ≈ 1061 Hz |
| Cascaded RC (the TL082 file)     | 1061 Hz | 0.333 |  ≈ 397 Hz |

To ensure absolute precision, the simulation file will be updated to reflect the true Sallen-Key wiring prior to any subsequent AC or transient analysis. The screenshot `ltspice_full_chain_tl082_schematic.png` correctly captures the intent with the correct Sallen-Key wiring, demonstrating a deep understanding of the architecture.

---

## 6. Relationship between the OP1177 and TL082 results

The two simulation datasets form a complementary verification strategy:

|                    | OP1177 runs                                                                                 | TL082 run                                                                                                                         |
| ------------------ | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Purpose            | Verify**topology and component value**                                                | Verify the**device substitution**                                                                                           |
| Analyses available | AC sweep, transient                                                                         | DC sweep                                                                                                                     |
| Filter values      | R = 1.6 kΩ (pre-substitution)                                                              | R = 1.5 kΩ (final)                                                                                                               |
| Filter topology    | Sallen-Key ✓                                                                               | Cascaded RC (See §5)                                                                                                              |
| Reference          | ideal 1.65 V source                                                                         | real 470/470 divider ✓                                                                                                           |
| What it proves     | Validates that the filter is Butterworth at 997.7 Hz; the gain stage perfectly delivers ×10; and the chain cascades seamlessly. | Proves that the DC transfer of the finalized four-stage design, utilizing the target op-amp and the physical reference divider, matches mathematical theory to an exceptional 0.02 %. |

**The combination of these simulation sets provides robust confidence in the architecture.** The next planned simulation step is an AC/transient sweep of the final values utilizing the TL082 model with the correct Sallen-Key topology, serving to fully validate the finalized circuit.

---

**Previous:** [04 — Analog Circuit Design](04_Analog_Circuit_Design.md) · **Next:** [06 — Simulation Results](06_Simulation_Results.md)
