# Day 1: Inception of Open-Source EDA, OpenLANE and the Sky130 PDK

## 1. Overview

This module introduces the foundations of open-source digital ASIC design. It covers how software is executed on hardware, the physical structure of an integrated circuit, the role of the Process Design Kit (PDK), and the RTL-to-GDSII flow. The practical session launches OpenLANE in interactive mode, prepares the `picorv32a` design and runs logic synthesis.

### Objectives
- Describe the physical anatomy of a packaged chip (package, pads, die, core, IP, macros).
- Explain how application software maps to hardware through the RISC-V ISA.
- Identify the inputs required for digital ASIC design: RTL, EDA tools and a PDK.
- Outline each stage of the RTL-to-GDSII flow and the tool that implements it in OpenLANE.
- Prepare a design, run synthesis and evaluate the results in OpenLANE.

### Prerequisites
- OpenLANE workshop environment (native or containerized)
- Sky130A PDK installed
- `picorv32a` design available under `designs/`

## Table of Contents
1. [Overview](#1-overview)
2. [Fundamentals: Chip Anatomy and RISC-V](#2-fundamentals-chip-anatomy-and-risc-v)
3. [SoC Design and OpenLANE](#3-soc-design-and-openlane)
4. [Laboratory: Interactive OpenLANE Flow](#4-laboratory-interactive-openlane-flow)
5. [Results](#5-results)
6. [Summary](#6-summary)
7. [Command Reference](#7-command-reference)

---

## 2. Fundamentals: Chip Anatomy and RISC-V

### 2.1 Package, Chip, Pads, Core and Die

| Term | Description |
|---|---|
| QFN-48 package | Quad Flat No-Lead package with 48 peripheral pins that encloses the silicon chip |
| Package pins | External electrical connections of the package |
| Wire bonds | Wires connecting package pins to the pads on the die |
| Pads | Peripheral die structures where wire bonds terminate. I/O driver circuits behind each pad buffer the core logic from external capacitive loads and provide ESD protection |
| Core | Central region of the die containing the synthesized logic cells, clock tree and functional blocks |
| Die | The complete silicon piece cut from the wafer: core, I/O pads, pad ring and boundary spacing |
| Foundry IP | Pre-designed, silicon-proven circuits (for example SRAM, ADC, DAC, PLL) that interface with the digital logic |
| Macro | Purely digital multi-gate functional block (for example a hardware multiplier or custom processor) placed in the core |

### 2.2 The RISC-V Instruction Set Architecture

RISC-V is an open-standard ISA based on reduced instruction set computing principles. The ISA is the abstract interface between software and hardware:

1. High-level code (C/C++) is translated into RISC-V assembly.
2. The assembly is encoded into 32-bit or 64-bit binary instructions.
3. The hardware pipeline (for example the PicoRV32 core) decodes each instruction and generates control signals for the logic gates, multiplexers and ALU.

### 2.3 From Application Software to Hardware

| Layer | Role |
|---|---|
| Application software | High-level programs running on top of an operating system |
| Operating system | Process memory, scheduling, system calls and device drivers |
| Compiler | Translates high-level code into target-specific assembly (RISC-V) |
| Assembler | Converts assembly into binary machine code |
| Hardware | The instruction decoder interprets the binary and drives physical switching of transistors |

**Illustrative example.** The statement `c = a + b;` compiles to `add a0, a1, a2`, which the assembler encodes as `0x00C58533`:

| funct7 | rs2 | rs1 | funct3 | rd | opcode |
|---|---|---|---|---|---|
| `0000000` | `01100` (a2) | `01011` (a1) | `000` | `01010` (a0) | `0110011` |

---

## 3. SoC Design and OpenLANE

### 3.1 Requirements for Open-Source Digital ASIC Design

| Requirement | Open-source examples |
|---|---|
| RTL | Verilog, SystemVerilog |
| EDA tools | Yosys, OpenROAD, Magic, KLayout, OpenSTA |
| PDK | SkyWater Sky130 |

The PDK is the interface between the foundry and the designer. It provides design rules (DRC), device models (SPICE), technology files, timing libraries (`.lib`) and standard-cell layouts (LEF/GDS). The SkyWater 130 nm PDK was jointly released by Google and SkyWater Foundry as a fully open-source PDK.

### 3.2 RTL-to-GDSII Flow

| Stage | Description |
|---|---|
| Synthesis | Translates RTL into a gate-level netlist mapped to standard-library cells |
| Floorplanning and power planning | Defines core and die dimensions, macro positions and pin layout; builds the power distribution network |
| Placement | Places standard cells in legal rows while minimizing wire length and congestion |
| Clock Tree Synthesis (CTS) | Builds a balanced clock distribution network with minimum skew and latency |
| Routing | Creates the physical metal interconnect according to netlist connectivity |
| Signoff | Static Timing Analysis (STA), Design Rule Checking (DRC) and Layout versus Schematic (LVS) verification |

### 3.3 OpenLANE

OpenLANE is an open-source, automated RTL-to-GDSII framework built around OpenROAD and other open-source EDA tools. The striVe family of open-source SoCs was created to validate and benchmark it.

| Stage | Tool |
|---|---|
| Synthesis | Yosys with abc |
| Floorplan and PDN | OpenROAD (`init_fp`, `pdngen`) |
| Placement | OpenROAD (`global_placement`, `detailed_placement`) |
| Clock tree synthesis | TritonCTS |
| Global routing | FastRoute |
| Detailed routing | TritonRoute |
| Signoff | Magic and KLayout (DRC, GDSII), Netgen (LVS), OpenSTA (timing) |

OpenLANE operates in two modes:
- **Automated mode:** executes the complete flow without interruption (`./flow.tcl -design picorv32a`).
- **Interactive mode:** executes each step individually for inspection and tuning (`./flow.tcl -interactive`).

---

## 4. Laboratory: Interactive OpenLANE Flow

### 4.1 Directory Structure

Root: `/work/tools/openlane_working_dir/openlane/`

```text
openlane/
├── flow.tcl          # flow entry point
├── designs/          # one sub-directory per design
│   └── picorv32a/
│       ├── config.tcl
│       ├── src/      # Verilog sources, SDC
│       └── runs/     # created by `prep`, one directory per run
├── pdks/             # Sky130 PDK: technology files, LEF, liberty, cell libraries
└── scripts/          # execution scripts and Tcl procedures
```

> The location of the PDK directory can differ between installations. Verify the path in your environment.

### 4.2 Launching OpenLANE and Preparing the Design

```bash
cd /work/tools/openlane_working_dir/openlane
./flow.tcl -interactive
```

If the environment is containerized, start the container before running these commands. At the `%` prompt:

```tcl
package require openlane 0.9
prep -design picorv32a
```

To assign a run name and overwrite a previous run:

```tcl
prep -design picorv32a -tag my_custom_run -overwrite
```

`prep` creates a run directory under `designs/picorv32a/runs/`:

```text
runs/<run_tag>/
├── config.tcl    # configuration used for this run
├── tmp/          # intermediate files (including merged.lef)
├── results/      # stage outputs
├── reports/      # stage reports
└── logs/         # stage logs
```

During preparation, the technology LEF (layer geometry) and the cell LEF (cell shapes and ports) are merged into `tmp/merged.lef`.

### 4.3 Running Synthesis

```tcl
run_synthesis
```

Yosys and abc map the RTL to the Sky130 standard-cell library. Typical outputs:

| Artifact | Location |
|---|---|
| Gate-level netlist | `runs/<run_tag>/results/synthesis/picorv32a.synthesis.v` |
| Synthesis statistics | `runs/<run_tag>/reports/synthesis/1-synthesis.stat.rpt` |

### 4.4 Reviewing the OpenLANE Repository

The official OpenLANE repository provides documentation, release history, feature flags, an issue tracker and the full list of configuration variables (`FP_*`, `SYNTH_*`, `CTS_*` and others) that can be set in `config.tcl` or with `set ::env(...)`.

---

## 5. Results

### 5.1 Synthesis Characterization

```bash
cd /work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/
cat <run_tag>/reports/synthesis/1-synthesis.stat.rpt
```

Metrics extracted from the report: total cell count, chip area and cell-type distribution.

$$\text{Flop Ratio} = \frac{\text{Number of D flip-flops}}{\text{Total number of cells}}$$

| Parameter | Value |
|---|---|
| Total cells | 14,876 |
| D flip-flops (`sky130_fd_sc_hd__dfxtp_2`) | 1,613 |
| Flop ratio | 1613 / 14876 ≈ 0.1084 (10.84 %) |

The values above are from the reference workshop run; results depend on tool version and configuration.

### 5.2 Evidence

| Figure | Description | File |
|---|---|---|
| 1 | OpenLANE interactive shell (`%` prompt) | `screenshots/01_interactive_shell.png` |
| 2 | Output of `prep -design picorv32a` | `screenshots/02_prep_output.png` |
| 3 | Generated run directory | `screenshots/03_run_directory.png` |
| 4 | Completion of `run_synthesis` | `screenshots/04_synthesis_complete.png` |
| 5 | Synthesis statistics report | `screenshots/05_stat_report.png` |
| 6 | Flop ratio calculation | `screenshots/06_flop_ratio.png` |

---

## 6. Summary

- A chip consists of a die (core and pads) enclosed in a package; wire bonds connect package pins to pads.
- The RISC-V ISA links software to hardware through compilation, assembly, decoding and execution.
- Open-source ASIC design requires RTL, EDA tools and a PDK.
- OpenLANE automates the RTL-to-GDSII flow and supports step-by-step execution in interactive mode.
- The flop ratio obtained after synthesis indicates the proportion of sequential elements in the design.

## 7. Command Reference

```bash
cd /work/tools/openlane_working_dir/openlane
./flow.tcl -interactive
```
```tcl
package require openlane 0.9
prep -design picorv32a
prep -design picorv32a -tag my_custom_run -overwrite   ;# optional: named run, overwrite existing
run_synthesis
```
```bash
cat designs/picorv32a/runs/<run_tag>/reports/synthesis/1-synthesis.stat.rpt
```
