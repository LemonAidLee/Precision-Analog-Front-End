# Documentation Index

Engineering documentation for the **Precision Mixed-Signal Data Acquisition Front End (AFE)**.

The repository landing page is **[../README.md](../README.md)**. This index is the entry point to the detailed technical documents.

Documents **01–10** and the **[Portfolio Summary](Portfolio_Summary.md)** form the comprehensive documentation suite. They detail the system architecture, component selection, simulation verification, and PCB layout that comprise the complete AFE design.

Every document here accurately reflects the current state of the design files. Calculations are explicitly shown, and simulation results directly reference the corresponding LTspice files. 

---

## Contents

| # | Document | What it covers |
|---|---|---|
| 01 | [Project Overview](01_Project_Overview.md) | The measurement problem, the engineering solution, and the resulting signal-conditioning architecture |
| 02 | [System Architecture](02_System_Architecture.md) | Block diagram, block-by-block contributions, power distribution, and grounding strategy |
| 03 | [Signal Path](03_Signal_Path.md) | Node-by-node walkthrough: VSENSOR → A → B → VFILTER → VAMP → VX → VADC |
| 04 | [Analog Circuit Design](04_Analog_Circuit_Design.md) | Per-stage topology, equations, component values, gain and headroom budgets |
| 05 | [LTspice Simulation](05_LTspice_Simulation.md) | Simulation strategy, file inventory, and TL082 macromodel integration |
| 06 | [Simulation Results](06_Simulation_Results.md) | Result tables, waveforms, and the electrical behavior established by simulation |
| 07 | [KiCad Schematic and PCB](07_KiCad_Schematic_and_PCB.md) | Schematic capture, board dimensions, layout rationale, and routing strategy |
| 08 | [Components and BOM](08_Components_and_BOM.md) | Comprehensive BOM by functional group, footprints, and selection criteria |
| 09 | [ESP32 + ADS1115 Interface](09_ESP32_ADS1115_Interface.md) | Connector pinouts, I²C integration, and ADC configuration |
| 10 | [Engineering Decision](10_Engineering_Decision.md) | Key architectural choices, alternatives considered, and quantitative reasoning |
| — | [Portfolio Summary](Portfolio_Summary.md) | High-level engineering overview and skills mapping |

> A further working document, `11_Engineering_Review.md`, is kept alongside this set in the project's local working copy. It consolidates the verification matrix and design-file consistency audit. It serves as an ongoing development log.

---

## Reference figures

**Schematic**
- [`images/schematic/schematic_full.svg`](images/schematic/schematic_full.svg) — complete Rev A schematic

**PCB**
- [`images/pcb/pcb_3d_view_isometric.png`](images/pcb/pcb_3d_view_isometric.png) — 3D render, isometric
- [`images/pcb/pcb_3d_view_top.png`](images/pcb/pcb_3d_view_top.png) — 3D render, top-down
- [`images/pcb/pcb_layout_2d.png`](images/pcb/pcb_layout_2d.png) — copper + silkscreen + outline
- [`images/pcb/pcb_layout_all_layers.svg`](images/pcb/pcb_layout_all_layers.svg) — vector, all plotted layers
- [`images/pcb/pcb_copper_top.svg`](images/pcb/pcb_copper_top.svg) / [`pcb_copper_bottom.svg`](images/pcb/pcb_copper_bottom.svg) — per-layer copper

**LTspice** — [`images/ltspice/`](images/ltspice/) holds figures that reflect the current design; [`images/ltspice/archive/`](images/ltspice/archive/) holds figures from superseded iterations, preserving the development history.

---

## Design files

Source files are **not duplicated** into `docs/`. They live in their working directories so the KiCad and LTspice projects stay intact:

| Domain | Location |
|---|---|
| KiCad 10 project | [`../KiCAD_PCB_Design/`](../KiCAD_PCB_Design/) |
| LTspice schematics, netlists, TL082 model | [`../LTSpice_Simulation/`](../LTSpice_Simulation/) |
| Original engineering log / working notes | [`../Precision DAQ AFE_Documentation/`](../Precision%20DAQ%20AFE_Documentation/) |

Generated exports (schematic PDF, PCB layout PDF, BOM CSV) are in [`design_files/`](design_files/) — see [`design_files/README.md`](design_files/README.md).

---

## How to read this if you have five minutes

1. [../README.md](../README.md) — what the project is and the current development status
2. [03 — Signal Path](03_Signal_Path.md) — how a signal actually moves through the AFE
3. [10 — Engineering Decision](10_Engineering_Decision.md) — why the circuit is structured the way it is
