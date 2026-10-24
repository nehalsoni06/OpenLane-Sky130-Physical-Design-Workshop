# Day 3: Design a Library Cell using Magic Layout and Characterize it with ngspice

## 1. Overview

This module walks through the design and characterization of a custom standard cell (a CMOS inverter). The layout is inspected in Magic, design rule violations are reviewed, parasitics are extracted into a SPICE netlist, and the cell is simulated in ngspice to measure propagation delay and transition time. The cell's LEF is then exported so that it can be used in the OpenLANE flow on Day 4.

### Objectives
- Describe the standard-cell design and characterization flow.
- Open and inspect a standard-cell layout in Magic and identify DRC violations.
- Apply the layout rules that make a cell compatible with the standard-cell library grid.
- Extract parasitic R and C from the layout into a SPICE netlist.
- Simulate the inverter in ngspice and measure delay and slew from the waveform.
- Export the cell's LEF file.

### Prerequisites
- Magic and ngspice installed
- Sky130A technology file and SPICE model files
- Git, to clone the cell design repository

## Table of Contents
1. [Overview](#1-overview)
2. [Standard-Cell Design and Characterization Flow](#2-standard-cell-design-and-characterization-flow)
3. [Layout Rules](#3-layout-rules)
4. [Timing Characterization Parameters](#4-timing-characterization-parameters)
5. [Laboratory: Inverter Layout, Extraction and Simulation](#5-laboratory-inverter-layout-extraction-and-simulation)
6. [Results](#6-results)
7. [Summary](#7-summary)
8. [Command Reference](#8-command-reference)

---

## 2. Standard-Cell Design and Characterization Flow

**Inputs:** PDK design rules, SPICE transistor models, and the target cell dimensions (library height and width constraints).

| Step | Description | Tool |
|---|---|---|
| 1 | Circuit design: schematic and SPICE netlist | SPICE |
| 2 | Layout design: transistor sizing, diffusion, substrate contacts, metal wiring | Magic |
| 3 | Parasitic extraction: R and C extracted from the layout into a SPICE netlist | Magic |
| 4 | SPICE simulation: transient simulation across process, voltage and temperature (PVT) corners | ngspice |
| 5 | Characterization: timing tables (`.lib`) for propagation delay, transition time and setup/hold windows | Characterization tool |

---

## 3. Layout Rules

For a cell to be usable by the placer and router, the layout must follow the library grid:

- Input and output ports must lie at the **intersections of the vertical and horizontal routing-grid tracks**.
- Cell **width** must be an integral multiple of the horizontal track pitch.
- Cell **height** must be an integral multiple of the vertical track pitch.

These rules ensure that routing tracks line up with cell pins and that cells tile cleanly in standard-cell rows.

---

## 4. Timing Characterization Parameters

| Parameter | Definition |
|---|---|
| Rise delay | $t_{out,50\%} - t_{in,50\%}$, for a falling input transition |
| Fall delay | $t_{out,50\%} - t_{in,50\%}$, for a rising input transition |
| Rise transition time | $t_{out,80\%} - t_{out,20\%}$ |
| Fall transition time | $t_{out,20\%} - t_{out,80\%}$ |

For an inverter, a falling input produces a rising output, and vice versa.

---

## 5.
