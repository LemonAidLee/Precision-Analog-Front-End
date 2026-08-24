# Design Files

[← Documentation index](../README.md)

---

## Source files — not duplicated here

The KiCad and LTspice projects are **not copied into `docs/`**. Duplicating them would create two versions of the truth and break KiCad's internal project references. They live in their working directories:

| Domain | Location | Contents |
|---|---|---|
| **KiCad 10** | [`../../KiCAD_PCB_Design/`](../../KiCAD_PCB_Design/) | `Precision_AFE_ESP32.kicad_pro` · `.kicad_sch` · `.kicad_pcb` · `.step` · `Precision_AFE_ESP32_drc_violations.json` |
| **LTspice** | [`../../LTSpice_Simulation/`](../../LTSpice_Simulation/) | `AFE_Integration_TL082.asc` / `.net` (current) · `TL082.lib` (model) · stage-development `.asc` files · `.raw` / `.log` results |
| **Engineering log** | [`../../Precision%20DAQ%20AFE_Documentation/`](../../Precision%20DAQ%20AFE_Documentation/) | Original working notes and the LTspice screenshot library |

> **Note on the engineering log:** `AFE_FINAL_KiCad_Reference.md` and `AFE_Master_Engineering_Documentation.md` were written during the design phase and are now out of date on several points (board size, TL082 verification status, reference circuit, power architecture, test points). They remain useful as a record of how the design was reasoned through. **`docs/` is the current description of the design.**

---

## Generated exports — in this directory

Produced with `kicad-cli` 10.0.5 from the current design files.

| File | What it is |
|---|---|
| `Precision_AFE_ESP32_schematic.pdf` | Complete Rev A schematic, A4, with title block |
| `Precision_AFE_ESP32_pcb_layout.pdf` | PCB layout — F.Cu, B.Cu, F.SilkS, Edge.Cuts |
| `Precision_AFE_ESP32_bom.csv` | Bill of materials, grouped by value and footprint |

---

## Not generated: fabrication output

**No Gerbers, drill files, or pick-and-place files exist yet.**

That used to be because seven nets on the board carried no copper — generating a fabrication package from a half-routed board would have produced an artefact that looked like a manufacturing release and was not one. That blocker is gone: routing is now complete (see [07 §5](../07_KiCad_Schematic_and_PCB.md#5-resolution--the-unrouted-nets-and-the-bug-behind-them)) and DRC is clean against a meaningful, fully-populated ratsnest. Gerbers simply haven't been generated yet — nothing about the current design file precludes it, but it remains a deliberate step for whoever decides this design is ready for a manufacturing release, not something to produce automatically alongside a documentation update.

---

## Which files are current

There is no ambiguity about versions:

| Concern | Answer |
|---|---|
| `KiCAD_PCB_Design/.history/` | KiCad's own Local History snapshot. **Byte-identical** to the top-level files — verified by diff. Not an older version. |
| `KiCAD_PCB_Design/.mcp-backups/` | Timestamped tool backups of the `.kicad_pcb`. Superseded. |
| `LTSpice_Simulation/*.asc` | Multiple files, all legitimate — one bench per design stage. `AFE_Integration_TL082.asc` is the current full-chain build; the rest are stage-development benches. See [05 §2](../05_LTspice_Simulation.md#2-file-inventory) for the full inventory with status. |
| `Draft2.*`, `AFE_SallenKey_OP1177.*` | Orphaned result files with no surviving `.asc`. Not cited anywhere. |

---

## Reproducing the exports

```bash
# Schematic PDF
kicad-cli sch export pdf \
  --output docs/design_files/Precision_AFE_ESP32_schematic.pdf \
  KiCAD_PCB_Design/Precision_AFE_ESP32.kicad_sch

# Schematic SVG (used in the documentation)
kicad-cli sch export svg \
  --output docs/images/schematic --exclude-drawing-sheet \
  KiCAD_PCB_Design/Precision_AFE_ESP32.kicad_sch

# PCB layout PDF
kicad-cli pcb export pdf \
  --output docs/design_files/Precision_AFE_ESP32_pcb_layout.pdf \
  --layers "F.Cu,B.Cu,F.SilkS,Edge.Cuts" --mode-single \
  KiCAD_PCB_Design/Precision_AFE_ESP32.kicad_pcb

# 3D renders
kicad-cli pcb render --output docs/images/pcb/pcb_3d_view_top.png \
  --width 1800 --height 1400 --side top --quality high \
  --background opaque --perspective --zoom 0.9 \
  KiCAD_PCB_Design/Precision_AFE_ESP32.kicad_pcb

kicad-cli pcb render --output docs/images/pcb/pcb_3d_view_isometric.png \
  --width 1800 --height 1400 --rotate "-30,0,25" --quality high \
  --background opaque --floor --zoom 0.85 \
  KiCAD_PCB_Design/Precision_AFE_ESP32.kicad_pcb

# BOM
kicad-cli sch export bom \
  --output docs/design_files/Precision_AFE_ESP32_bom.csv \
  --fields "Reference,Value,Footprint,\${QUANTITY}" \
  --group-by "Value,Footprint" \
  KiCAD_PCB_Design/Precision_AFE_ESP32.kicad_sch
```

---

## Running the simulation

Open `LTSpice_Simulation/AFE_Integration_TL082.asc` in LTspice 24.x and run.

> ⚠️ The `.lib` directive on that schematic uses an **absolute Windows path** to `TL082.lib`. On any other machine, or after moving the project, change it to a bare `TL082.lib` — the model file sits in the same directory.

> ⚠️ That file's filter section does not match the KiCad schematic — both capacitors return to ground, making it a passive 2-pole RC rather than a Sallen-Key. **DC results are unaffected**; AC and transient results from it would not represent the board. See [05 — LTspice Simulation §5](../05_LTspice_Simulation.md#5-open-issue--filter-topology-in-the-tl082-build).
