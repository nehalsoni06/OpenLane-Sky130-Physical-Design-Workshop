# Day 4: Pre-Layout Timing Analysis and the Importance of a Good Clock Tree

## 1. Overview

This module covers how gate delay is modelled with lookup tables, how timing is analysed with ideal clocks in OpenSTA, and why a balanced clock tree matters. It then introduces clock tree synthesis with TritonCTS and signal-integrity techniques (buffering and shielding). The practical session integrates the custom inverter from Day 3 into the OpenLANE flow, runs CTS, and repeats timing analysis with real (propagated) clocks.

### Objectives
- Explain how input slew and output load determine gate delay, and how `.lib` lookup tables capture this.
- Compute setup and hold slack with ideal clocks.
- Describe H-tree clock distribution, clock buffering, crosstalk and shielding.
- Integrate a custom cell (LEF and liberty files) into OpenLANE and re-run synthesis, floorplan and placement.
- Run CTS, verify its outputs and evaluate timing with real clocks.

### Prerequisites
- Completed Days 1 to 3 (design prepared, `sky130_vsdinv.lef` exported)
- Liberty files for the custom cell and the Sky130 typical, fast and slow corners
- OpenSTA available as the `sta` binary (or OpenROAD with its STA commands)

## Table of Contents
1. [Overview](#1-overview)
2. [Timing Modelling Using Delay Tables](#2-timing-modelling-using-delay-tables)
3. [Timing Analysis with Ideal Clocks](#3-timing-analysis-with-ideal-clocks)
4. [Clock Tree Synthesis and Signal Integrity](#4-clock-tree-synthesis-and-signal-integrity)
5. [Timing Analysis with Real Clocks](#5-timing-analysis-with-real-clocks)
6. [Laboratory: Custom Cell Integration, CTS and STA](#6-laboratory-custom-cell-integration-cts-and-sta)
7. [Results](#7-results)
8. [Summary](#8-summary)
9. [Command Reference](#9-command-reference)

---

## 2. Timing Modelling Using Delay Tables

The propagation delay of a gate depends on two quantities:

| Factor | Effect |
|---|---|
| Input slew (transition time) | A slower input transition increases gate delay |
| Output load capacitance | A larger load takes longer to charge and discharge |

Characterized `.lib` files store this relationship as two-dimensional **lookup tables**: each combination of input slew and output capacitance maps to an output delay and an output slew. Values between table points are interpolated by the timing engine.

---

## 3. Timing Analysis with Ideal Clocks

An **ideal clock** assumes zero clock-network delay and zero skew at every sequential sink. Analysis at this stage uses only the combinational and register delays.

### 3.1 Setup Check

$$T_{required} = T_{clk} - T_{setup} - T_{uncertainty}$$

$$T_{arrival} = T_{clk\text{-}to\text{-}q} + T_{comb,max}$$

$$\text{Setup slack} = T_{required} - T_{arrival} \ge 0$$

### 3.2 Hold Check

$$T_{arrival} = T_{clk\text{-}to\text{-}q} + T_{comb,min}$$

$$T_{required} = T_{hold} + T_{uncertainty}$$

$$\text{Hold slack} = T_{arrival} - T_{required} \ge 0$$

A negative slack indicates a violation: negative setup slack means the path is too slow for the clock period; negative hold slack means data changes too soon after the capture edge.

---

## 4. Clock Tree Synthesis and Signal Integrity

### 4.1 Clock Tree Routing and Buffering (H-Tree)

The objective of CTS is to deliver the clock to every sequential element with minimum skew and insertion delay:

$$\text{Skew} = t_{clk,launch} - t_{clk,capture}$$

| Concept | Description |
|---|---|
| H-tree topology | Balanced, symmetric branching that gives every flip-flop sink the same path length from the clock root |
| Clock buffers | Buffers inserted along the tree to preserve signal transition speed and balance delay |

### 4.2 Crosstalk and Clock Net Shielding

| Concept | Description |
|---|---|
| Crosstalk | Switching transients on a net couple capacitively into neighbouring nets (aggressor and victim) |
| Shielding | Grounded ($V_{SS}$) or supply ($V_{DD}$) wires routed alongside critical clock nets to contain the electric field and reduce coupling |

Clock nets are the most critical because they switch every cycle and drive every sequential element; coupling-induced delay variation on them directly translates into skew.

---

## 5. Timing Analysis with Real Clocks

After CTS, ideal clocks are replaced by the **actual clock distribution delays** produced by TritonCTS. Setup and hold slack are then evaluated with real clock latency and skew on both the launching and capturing paths. Hold violations typically appear or worsen at this stage and are addressed by delay-buffer insertion during timing optimization.

---

## 6. Laboratory: Custom Cell Integration, CTS and STA

### 6.1 Copy the Custom Cell Files into the Design

```bash
cd /work/tools/openlane_working_dir/vsdstdcelldesign
cp sky130_vsdinv.lef /work/tools/openlane_working_dir/openlane/designs/picorv32a/src/
cp libs/sky130_fd_sc_hd__*.lib /work/tools/openlane_working_dir/openlane/designs/picorv32a/src/
```

The liberty files copied here are the typical, fast and slow files that include the custom inverter cell.

### 6.2 Configure OpenLANE and Re-run the Flow

In the OpenLANE interactive shell, after `prep -design picorv32a -tag <run_tag> -overwrite`:

```tcl
# Point synthesis and timing at the liberty files that include the custom cell
set ::env(LIB_SYNTH)   $::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib
set ::env(LIB_FASTEST) $::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib
set ::env(LIB_SLOWEST) $::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib
set ::env(LIB_TYPICAL) $::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib

# Add the extra LEF to the merged LEF used by the flow
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs

run_synthesis
run_floorplan
run_placement
```

> Variable names and the exact way to add LEFs differ between OpenLANE versions. Confirm against the documentation for your release.

After synthesis, confirm the custom cell appears in the netlist and in `tmp/merged.lef`:

```bash
grep -c sky130_vsdinv designs/picorv32a/runs/<run_tag>/results/synthesis/picorv32a.synthesis.v
grep -c sky130_vsdinv designs/picorv32a/runs/<run_tag>/tmp/merged.lef
```

### 6.3 Pre-Layout STA (Ideal Clocks)

Create the constraint file `designs/picorv32a/src/my_base.sdc`:

```tcl
create_clock -name clk -period 12.000 [get_ports clk]
set_input_delay  -clock clk 2.500 [all_inputs]
set_output_delay -clock clk 2.500 [all_outputs]
set_max_fanout 16 [current_design]
```

Run OpenSTA (`sta pre_sta.conf`, or paste the commands into the `sta` shell):

```tcl
set WD /work/tools/openlane_working_dir

read_liberty -max $WD/pdks/sky130A/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__slow.lib
read_liberty -min $WD/pdks/sky130A/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__fast.lib
read_verilog $WD/openlane/designs/picorv32a/runs/<run_tag>/results/synthesis/picorv32a.synthesis.v
link_design picorv32a
read_sdc $WD/openlane/designs/picorv32a/src/my_base.sdc

report_checks -path_delay max -fields {slew cap input_pins fanout} -digits 4
report_check
