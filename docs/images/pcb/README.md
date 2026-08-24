# PCB and Schematic Figures

[← Documentation index](../../README.md)

All figures generated with `kicad-cli` 10.0.5 from `KiCAD_PCB_Design/Precision_AFE_ESP32.kicad_pcb` / `.kicad_sch` as they exist on disk. Commands are listed in [`../../design_files/README.md`](../../design_files/README.md).

**These are renders and plots of a design file. No physical board exists.**

---

## PCB

| File | Contents |
|---|---|
| `pcb_3d_view_isometric.png` | 3D render, isometric, with component models — including the ESP32 DevKit and ADS1115 breakout as they would seat in their sockets |
| `pcb_3d_view_top.png` | 3D render, top-down. Best view of the analog/digital split and the silkscreen labelling |
| `pcb_layout_2d.png` | F.Cu (red), B.Cu (blue), F.SilkS and Edge.Cuts, colour plot. Fully routed; the near-solid blue field is the GND pour on B.Cu fusing visually with the B.Cu signal traces it shares a net with |
| `pcb_layout_all_layers.svg` | Same layer set plus B.SilkS/F.Mask/B.Mask, vector |
| `pcb_copper_top.svg` | F.Cu only, with board outline |
| `pcb_copper_bottom.svg` | B.Cu only, mirrored, with board outline |

**Board:** 78.1 mm × 99.6 mm (bounding box) · 2 layers · 1.6 mm · 29 through-hole footprints · 209 track segments · 10 vias · GND copper pour on both layers.

## Schematic

| File | Contents |
|---|---|
| [`../schematic/schematic_full.svg`](../schematic/schematic_full.svg) | Complete Rev A schematic, single A4 sheet, drawing sheet excluded |

A PDF version with the title block is at [`../../design_files/Precision_AFE_ESP32_schematic.pdf`](../../design_files/Precision_AFE_ESP32_schematic.pdf).
