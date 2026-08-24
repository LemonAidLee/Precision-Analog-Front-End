# LTspice Figures

[← Documentation index](../../README.md)

Screenshots captured from LTspice during design and verification. Originals live in [`../../../Precision%20DAQ%20AFE_Documentation/LTSpice%20Simulation%20Image/`](../../../Precision%20DAQ%20AFE_Documentation/) under their original names; the copies here have been renamed to describe their contents and to be filesystem- and URL-safe.

**Current design point:** R1 = R2 = 1.5 kΩ, Rin1 = 36 kΩ, TL082.

---

## Figures reflecting the current design

| File | Contents | Design point |
|---|---|---|
| `ltspice_full_chain_tl082_schematic.png` | Four-stage chain drawn with TL082 and the real 470/470 reference divider, with stage annotations | R = 1.5 kΩ, Rin1 = 36 kΩ, TL082 |
| `ltspice_full_chain_op1177_reference_schematic.png` | The same four-stage chain drawn with OP1177 and an ideal 1.65 V source — the reference bench | R = 1.5 kΩ, Rin1 = 36 kΩ, OP1177 |

> The circuit in `ltspice_full_chain_op1177_reference_schematic.png` corresponds to `AFE_IntegrationComplete.asc`, which **no longer runs** in its saved state.

## Filter characterisation *(R = 1.6 kΩ — one design point behind the board)*

| File | Contents |
|---|---|
| `ltspice_sallenkey_bode_response.png` | Magnitude and phase, 10 Hz – 100 kHz. Passband +4.03 dB, corner ≈ 998 Hz, 40 dB/decade roll-off |
| `ltspice_sallenkey_transient_100hz.png` | 100 Hz: 1 V in → 1.59 V out, in phase (deep in the passband) |
| `ltspice_sallenkey_transient_1khz.png` | 1 kHz: 1 V in → 1.13 V out with visible phase lag (at the corner) |
| `ltspice_sallenkey_transient_10khz.png` | 10 kHz: heavy attenuation (a decade above the corner) |

## Gain stage

| File | Contents |
|---|---|
| `ltspice_gain_stage_schematic.png` | Non-inverting ×10: Rf = 90 kΩ, Rg = 10 kΩ, ±12 V, OP1177 |
| `ltspice_gain_stage_ac_response.png` | AC response: 20 dB flat through 10 kHz, corner ≈ 100 kHz |
| `ltspice_gain_stage_transient_100hz.png` | 100 Hz: 100 mV in → 1.0 V out |
| `ltspice_gain_stage_transient_1khz.png` | 1 kHz: 100 mV in → 1.0 V out |

## Filter + gain cascade *(R = 1.6 kΩ)*

VSENSOR (green) / VFILTER (blue) / VAMP (red) on one plot:

| File | Excitation | VFILTER | VAMP |
|---|---|---|---|
| `ltspice_filter_gain_chain_transient_100hz.png` | 100 mV, 100 Hz | 159 mV | 1.59 V |
| `ltspice_filter_gain_chain_transient_500hz.png` | 100 mV, 500 Hz | 154 mV | 1.54 V |
| `ltspice_filter_gain_chain_transient_1khz.png` | 100 mV, 1 kHz | 113 mV | 1.13 V |
| `ltspice_filter_gain_chain_transient_10khz.png` | 100 mV, 10 kHz | 1.5 mV | 15 mV |

Each of these agrees with the independently-run AC sweep to three significant figures.

---

## Archive

[`archive/`](archive/) holds figures from **superseded design points**. They are kept for traceability of how the design evolved, and every filename carries the design point it belongs to. Do not quote numbers from them as current.

| File | Why it is archived |
|---|---|
| `ltspice_sallenkey_ac_testbench_r1k6.png` | Filter AC test bench schematic at R = 1.6 kΩ (board uses 1.5 kΩ) |
| `ltspice_sallenkey_transient_testbench_r1k6.png` | Filter transient test bench schematic at R = 1.6 kΩ |
| `ltspice_level_shifter_testbench_rin20k.png` | Level-shifter bench with **Rin = 20 kΩ** — the rejected design point (gain 1.5) |
| `ltspice_level_shifter_transient_rin20k.png` | Its waveform: ±1 V in → 0.15…3.15 V out. Demonstrates the *principle* correctly; the numbers do not apply to the current 36 kΩ design |
| `ltspice_full_chain_op1177_earlier_r1k6_rin20k.png` | Earlier four-stage chain at R = 1.6 kΩ **and** Rin1 = 20 kΩ — two design points behind |

---

## Name mapping

| Original filename | Renamed to |
|---|---|
| `AFE_TL082_FinalIntegradedCircuit.png` | `ltspice_full_chain_tl082_schematic.png` |
| `AFE_FinalCompleteCircuit_AllValuesFinalized.png` | `ltspice_full_chain_op1177_reference_schematic.png` |
| `Integration_Finalized.png` | *(byte-identical duplicate of the above — not copied)* |
| `BodePlot.png` | `ltspice_sallenkey_bode_response.png` |
| `Transient_100Hz.png` / `_1kHz` / `_10kHz` | `ltspice_sallenkey_transient_100hz.png` / `_1khz` / `_10khz` |
| `AFE_Amplifier_Topology.png` | `ltspice_gain_stage_schematic.png` |
| `Amplifie_AcGain&Bandwidth.png` | `ltspice_gain_stage_ac_response.png` |
| `Amplifier_100Hz.png` / `Amplifier_1kHz.png` | `ltspice_gain_stage_transient_100hz.png` / `_1khz` |
| `Integration_100Hz.png` / `_500Hz` / `_1kHz` / `_10kHz` | `ltspice_filter_gain_chain_transient_100hz.png` / `_500hz` / `_1khz` / `_10khz` |
| `AFE_SallenKey_OP1177_BodePlotAnalysis.png` | `archive/ltspice_sallenkey_ac_testbench_r1k6.png` |
| `AFE_SallenKey_OP1177_TransientAnalysis.png` | `archive/ltspice_sallenkey_transient_testbench_r1k6.png` |
| `AFE_LevelShifter_Circuit.png` | `archive/ltspice_level_shifter_testbench_rin20k.png` |
| `TransferEquation_Waveform.png` | `archive/ltspice_level_shifter_transient_rin20k.png` |
| `AFE_CompleteIntegratedCircuit.png` | `archive/ltspice_full_chain_op1177_earlier_r1k6_rin20k.png` |

Two original names carried errors that the renaming corrects: `FinalIntegradedCircuit` (typo) and `Amplifie_AcGain&Bandwidth` (typo, plus an `&` that is awkward in URLs and shells). The two `..._BodePlotAnalysis` / `..._TransientAnalysis` names described *plots* but the files are *schematics* — the new names say which.

**The original files were not modified, renamed, or deleted.** These are copies.
