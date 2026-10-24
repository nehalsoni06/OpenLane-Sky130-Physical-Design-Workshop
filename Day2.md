# Day 2: Good Floorplan vs Bad Floorplan and Introduction to Library Cells

## 1. Overview

This module covers the decisions that define a chip floorplan (core utilization, aspect ratio, pre-placed cells, decoupling capacitors, power planning and pin placement), followed by library binding, placement, and the characterization of standard cells. The practical session runs floorplanning and placement for `picorv32a` in OpenLANE and inspects the results in Magic.

### Objectives
- Define core and die area, utilization factor and aspect ratio, and compute them for a given floorplan.
- Explain the purpose of pre-placed cells, decoupling capacitors, power planning and placement blockages.
- Run floorplanning and placement in OpenLANE and control them through configuration variables.
- Inspect floorplan and placement DEF files in the Magic layout viewer.
- Describe how a standard cell is characterized and how delay and slew are measured.

### Prerequisites
- Completed Day 1 (design prepared with `prep -design picorv32a`, synthesis finished)
- Magic layout viewer and the Sky130A technology file

## Table of Contents
1. [Overview](#1-overview)
2. [Chip Floorplanning Considerations](#2-chip-floorplanning-considerations)
3. [Library Binding and Placement](#3-library-binding-and-placement)
4. [Cell Design and Characterization Flow](#4-cell-design-and-characterization-flow)
5. [Timing Characterization Parameters](#5-timing-characterization-parameters)
6. [Laboratory: Floorplan and Placement in OpenLANE](#6-laboratory-floorplan-and-placement-in-openlane)
7. [Results](#7-results)
8. [Summary](#8-summary)
9. [Command Reference](#9-command-reference)

---

## 2. Chip Floorplanning Considerations

### 2.1 Utilization Factor and Aspect Ratio

| Term | Description |
|---|---|
| Core area | Enclosed area available for placing standard logic cells |
| Die area | Total area including the core, pads and boundary clearance |

$$\text{Utilization Factor} = \frac{\text{Area occupied by standard logic cells}}{\text{Total core area}}$$

$$\text{Aspect Ratio} = \frac{\text{Core height}}{\text{Core width}}$$

**Example.** Core = 100 µm × 100 µm = 10,000 µm². Logic cells occupy 5,000 µm².

$$\text{Utilization} = \frac{5000}{10000} = 0.5 \;(50\%)$$

- A utilization of 1.0 (100 %) leaves no space for routing or for buffers inserted during clock tree synthesis and optimization. Practical designs target roughly 50–70 %.
- An aspect ratio of 1.0 describes a square core.

### 2.2 Pre-placed Cells

Complex blocks (for example SRAM, MAC units, PLLs) are positioned manually before the automated placement step. They are locked in place and surrounded by placement blockages so that the placer does not overlap them.

### 2.3 Decoupling Capacitors

Switching events cause supply fluctuations and temporary logic errors:

$$\Delta V = L \cdot \frac{di}{dt}$$

Decoupling capacitors are placed next to pre-placed blocks and power rails. They store local charge and supply instantaneous switching current, which reduces IR drop and supply noise.

### 2.4 Power Planning

Long power rails carrying current develop a resistive drop:

$$V_{drop} = I \cdot R$$

A Power Distribution Network (PDN) addresses this using top-level metal layers: power rings around the core, plus horizontal and vertical $V_{DD}$ and $V_{SS}$ straps that form a low-resistance mesh across the chip.

### 2.5 Pin Placement and Logical Cell Placement Blockage

- **Pin placement:** I/O pins are distributed along the core boundary according to timing paths and the external pinout.
- **Placement blockage:** physical regions around I/O borders and macro boundaries in which the tool may not place standard cells.

---

## 3. Library Binding and Placement

**Library binding** maps the gates in the netlist (AND, OR, flip-flops, ...) to physical standard cells defined in the `.lib` (timing) and `.lef` (geometry) libraries.

| Placement step | Description |
|---|---|
| Global placement | Spreads cells across the floorplan to minimize total wire length; positions are not yet legal |
| Detailed placement | Adjusts cells to align on standard-cell rows and removes overlaps |

---

## 4. Cell Design and Characterization Flow

**Inputs:** PDK rules, SPICE transistor models and layout target dimensions.

| Step | Description |
|---|---|
| 1 | Schematic entry and SPICE netlist generation |
| 2 | Physical layout (mask) design in Magic |
| 3 | Parasitic extraction (R and C) to obtain an interconnect SPICE model |
| 4 | SPICE simulation with ngspice across process, voltage and temperature (PVT) corners |
| 5 | Generation of `.lib` timing tables |

---

## 5. Timing Characterization Parameters

| Parameter | Definition |
|---|---|
| Rise delay | $t_{out,50\%} - t_{in,50\%}$, for a falling input transition |
| Fall delay | $t_{out,50\%} - t_{in,50\%}$, for a rising input transition |
| Rise transition time | $t_{out,80\%} - t_{out,20\%}$ |
| Fall transition time | $t_{out,20\%} - t_{out,80\%}$ |

---

## 6. Laboratory: Floorplan and Placement in OpenLANE

### 6.1 Floorplan Configuration

Floorplan behaviour is controlled by variables in `designs/picorv32a/config.tcl`. They can also be overridden in the interactive shell before running the step:

```tcl
set ::env(FP_CORE_UTIL) 50          ;# core utilization (%)
set ::env(FP_ASPECT_RATIO) 1        ;# core height / width (1 = square)
set ::env(FP_CORE_MARGIN) 10        ;# margin between core and die (um)
set ::env(FP_IO_HMETRIC) 1          ;# horizontal I/O pin spacing metric
set ::env(FP_IO_VMETRIC) 1          ;# vertical I/O pin spacing metric
```

PDN strap spacing is set with `FP_PDN_VPITCH` and `FP_PDN_HPITCH`.

| Variable | Meaning |
|---|---|
| `FP_CORE_UTIL` | Core utilization (%) |
| `FP_ASPECT_RATIO` | Core height / core width |
| `FP_CORE_MARGIN` | Spacing between core and die boundary |
| `FP_IO_HMETRIC`, `FP_IO_VMETRIC` | I/O pin placement spacing |
| `FP_PDN_VPITCH`, `FP_PDN_HPITCH` | Vertical / horizontal power strap pitch |

### 6.2 Running Floorplan

```tcl
run_floorplan
```

Output: `runs/<run_tag>/results/floorplan/picorv32a.floorplan.def`

### 6.3 Viewing the Floorplan in Magic

Magic requires the technology file and the merged LEF to interpret the DEF.

```bash
cd /work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/<run_tag>/results/floorplan/

magic -T /work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.lef \
      def read picorv32a.floorplan.def &
```

> Adjust the technology-file and `merged.lef` paths to match your installation and run directory.

| Magic action | Key / command |
|---|---|
| Zoom in | `z` |
| Zoom out | `Z` |
| Fit design to window | `v` |
| Select object under cursor | `s` (cursor over object) |
| Identify selected object | `what` (in the Magic console) |

**Determining the die area from the DEF.** Read the `UNITS DISTANCE MICRONS` and `DIEAREA` entries:

```bash
grep -E "UNITS DISTANCE MICRONS|DIEAREA" picorv32a.floorplan.def
```

Divide the DIEAREA coordinates by the units-per-micron value to obtain dimensions in micrometres, then compute width × height.

### 6.4 Running Placement

```tcl
run_placement
```

Global placement followed by detailed placement. Output: `runs/<run_tag>/results/placement/picorv32a.placement.def`

### 6.5 Viewing the Placement in Magic

```bash
cd /work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/<run_tag>/results/placement/

magic -T /work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.lef \
      def read picorv32a.placement.def &
```

Verify that standard cells sit on legal rows, do not overlap, and are placed within the core.

---

## 7. Results

### 7.1 Floorplan Summary

| Parameter | Value |
|---|---|
| Core utilization (`FP_CORE_UTIL`) | _fill in_ |
| Aspect ratio (`FP_ASPECT_RATIO`) | _fill in_ |
| Die width × height (µm) | _fill in_ |
| Die area (µm²) | _fill in_ |

### 7.2 Evidence

| Figure | Description | File |
|---|---|---|
| 1 | Output of `run_floorplan` | `screenshots/01_run_floorplan.png` |
| 2 | Floorplan DEF opened in Magic | `screenshots/02_floorplan_magic.png` |
| 3 | `DIEAREA` calculation from the DEF | `screenshots/03_die_area.png` |
| 4 | Output of `run_placement` | `screenshots/04_run_placement.png` |
| 5 | Placement DEF opened in Magic | `screenshots/05_placement_magic.png` |
| 6 | Zoomed view of cells on rows | `screenshots/06_cells_on_rows.png` |

---

## 8. Summary

- Utilization is the ratio of cell area to core area; an aspect ratio of 1.0 gives a square core.
- Pre-placed cells are fixed before placement and protected by blockages.
- Decoupling capacitors and a power mesh limit supply noise and IR drop.
- Pins are placed on the core boundary based on connectivity and timing.
- Placement proceeds from global (wire-length driven) to detailed (legalized on rows).
- Cell characterization relies on extracted SPICE simulation; delay is measured at 50 % points and slew between 20 % and 80 %.

## 9. Command Reference

```tcl
# OpenLANE interactive shell (after `prep -design picorv32a` and `run_synthesis`)
set ::env(FP_CORE_UTIL) 50
set ::env(FP_ASPECT_RATIO) 1
run_floorplan
run_placement
```
```bash
# View in Magic (from results/floorplan or results/placement)
magic -T $PDK/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.lef \
      def read picorv32a.floorplan.def &
```
