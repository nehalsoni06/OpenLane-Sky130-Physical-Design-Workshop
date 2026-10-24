# Day 5: Final Steps for RTL to GDSII using TritonRoute and OpenSTA

## 1. Overview

This module completes the RTL-to-GDSII flow. It covers the power distribution network (PDN) generation, global and detailed routing with TritonRoute, parasitic extraction to SPEF, post-route static timing analysis, and generation of the final GDSII layout. The practical session routes `picorv32a`, extracts parasitics, verifies timing with the extracted parasitics and views the finished layout.

### Objectives
- Generate the power distribution network before routing.
- Distinguish global routing (FastRoute) from detailed routing (TritonRoute) and state the design rules the router must obey.
- Run routing in OpenLANE and verify that it finishes without DRC violations.
- Extract parasitics into a SPEF file and use it for post-route timing analysis.
- Generate and inspect the final GDSII layout.

### Prerequisites
- Completed Days 1 to 4 (synthesis, floorplan, placement and CTS finished in the same run)
- SPEF extractor tool available in the working directory
- OpenSTA, Magic and KLayout installed

## Table of Contents
1. [Overview](#1-overview)
2. [Routing Concepts](#2-routing-concepts)
3. [Post-Route Parasitics and Timing](#3-post-route-parasitics-and-timing)
4. [GDSII Generation](#4-gdsii-generation)
5. [Laboratory: PDN, Routing, SPEF, STA and GDSII](#5-laboratory-pdn-routing-spef-sta-and-gdsii)
6. [Results](#6-results)
7. [Summary](#7-summary)
8. [Command Reference](#8-command-reference)

---

## 2. Routing Concepts

### 2.1 Global Routing (FastRoute)

The layout is divided into a grid of three-dimensional routing regions (G-cells). FastRoute assigns every net a coarse path through these regions, balancing congestion. No physical metal is placed at this stage. The output is a set of **route guides**.

### 2.2 Detailed Routing (TritonRoute)

TritonRoute places the actual metal tracks and vias inside the route guides while strictly obeying the design rules of the technology:

| Rule class | Description |
|---|---|
| Minimum width and spacing | Smallest allowed metal width and spacing between shapes |
| Via enclosure | Required metal overlap around vias |
| Parallel run length | Spacing that grows with the length over which two wires run side by side |

Main steps: route-guide pre-processing, panel-based track assignment, inter-guide connectivity, and iterative rip-up and re-route until DRC violations are eliminated. The target is **zero DRC violations**.

---

## 3. Post-Route Parasitics and Timing

After routing, the real wire lengths and neighbours are known. **Parasitic extraction** computes the resistance (R) and capacitance (C) of every net and writes them in the Standard Parasitic Exchange Format (**SPEF**).

Running STA with the SPEF included gives the final, signoff-quality setup and hold slack, because wire delay and coupling are now accounted for. The earlier ideal-clock and post-CTS analyses used estimates.

$$\text{Setup slack} = T_{required} - T_{arrival} \ge 0 \qquad \text{Hold slack} = T_{arrival} - T_{required} \ge 0$$

---

## 4. GDSII Generation

The routed layout (DEF with the LEF views) is converted into a binary **GDSII** stream file. GDSII contains every mask layer needed for fabrication. In OpenLANE the conversion is performed by Magic; the result can be inspected in KLayout or Magic.

---

## 5. Laboratory: PDN, Routing, SPEF, STA and GDSII

### 5.1 Generate the Power Distribution Network

The PDN must exist before routing. In the OpenLANE interactive shell, after `run_cts`:

```tcl
gen_pdn
```

This builds the power rings and the $V_{DD}$/$V_{SS}$ straps and rails over the placed design. Check the log for errors before continuing.

### 5.2 Run Routing

Optional routing parameters can be set first:

```tcl
set ::env(ROUTING_OPT_ITERS) 64     ;# detailed-routing optimization iterations
set ::env(GLB_RT_ADJUSTMENT) 0.15   ;# fraction of routing capacity reserved by global routing
```

Then run global and detailed routing:

```tcl
run_routing
```

Outputs:

| Artifact | Location |
|---|---|
| Routed DEF | `runs/<run_tag>/results/routing/picorv32a.def` |
| Routing logs | `runs/<run_tag>/logs/routing/` |

Review the detailed-routing log: the iteration summary should end with **0 violations**. If violations remain, increase `ROUTING_OPT_ITERS` or lower the utilization and run again.

> Variable names and defaults differ between OpenLANE versions. Confirm them in the documentation for your release.

### 5.3 Extract Parasitics (SPEF)

```bash
cd /work/tools/openlane_working_dir/SPEF_Extractor

python3 main.py \
  /work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/<run_tag>/tmp/merged.lef \
  /work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/<run_tag>/results/routing/picorv32a.def
```

This produces `picorv32a.spef` next to the routed DEF in `results/routing/`.

### 5.4 Post-Route Timing Analysis with SPEF

Run OpenSTA (`sta`) with the same constraint file used on Day 4:

```tcl
set WD /work/tools/openlane_working_dir
set RUN $WD/openlane/designs/picorv32a/runs/<run_tag>

read_liberty $WD/pdks/sky130A/libs.ref/sky130_fd_sc_hd/lib/sky130_fd_sc_hd__typical.lib
read_verilog $RUN/results/synthesis/picorv32a.synthesis.v
link_design picorv32a

read_sdc $WD/openlane/designs/picorv32a/src/my_base.sdc
read_spef $RUN/results/routing/picorv32a.spef
set_propagated_clock [all_clocks]

report_checks -path_delay max -format full_clock_expanded
report_checks -path_delay min -format full_clock_expanded
```

> Use the netlist your run produced after CTS and routing if your flow writes one under `results/`. The synthesis netlist is used here only if no later netlist is available.

Record the worst setup and hold slack. Both should be non-negative for a design that meets timing.

### 5.5 Generate the GDSII Layout

```tcl
run_magic
```

Output: `runs/<run_tag>/results/magic/picorv32a.gds`

### 5.6 View the Final Layout

```bash
klayout /work/tools/openlane_working_dir/openlane/designs/picorv32a/runs/<run_tag>/results/magic/picorv32a.gds
```

Inspect the metal layers, the power straps and the routed nets. Alternatively, open the GDS in Magic.

---

## 6. Results

### 6.1 Routing Summary

| Parameter | Value |
|---|---|
| Global routing completed | _fill in_ |
| Final DRC violations (detailed routing) | _fill in_ |
| `ROUTING_OPT_ITERS` | _fill in_ |

### 6.2 Final Timing Summary

| Stage | Clock | Worst setup slack (ns) | Worst hold slack (ns) |
|---|---|---|---|
| Pre-layout | Ideal | _fill in_ | _fill in_ |
| Post-CTS | Propagated | _fill in_ | _fill in_ |
| Post-route (with SPEF) | Propagated | _fill in_ | _fill in_ |

### 6.3 Evidence

| Figure | Description | File |
|---|---|---|
| 1 | `gen_pdn` output | `screenshots/01_gen_pdn.png` |
| 2 | `run_routing` output | `screenshots/02_run_routing.png` |
| 3 | Detailed-routing log showing 0 violations | `screenshots/03_drc_clean.png` |
| 4 | SPEF extraction command and generated file | `screenshots/04_spef.png` |
| 5 | Post-route setup and hold reports | `screenshots/05_post_route_sta.png` |
| 6 | `run_magic` output | `screenshots/06_run_magic.png` |
| 7 | Final GDSII in KLayout | `screenshots/07_final_gds.png` |

---

## 7. Summary

- Global routing plans coarse paths through G-cells; detailed routing places the actual metal and vias under the design rules.
- The PDN is generated before routing, and routing is iterated until there are zero DRC violations.
- SPEF captures the real R and C of the routed nets and enables signoff-quality timing analysis.
- Final setup and hold slack must be checked with the parasitics included.
- The GDSII file is the mask-level output of the flow and is the handoff to fabrication.

## 8. Command Reference

```tcl
# OpenLANE interactive shell (after run_cts)
gen_pdn
set ::env(ROUTING_OPT_ITERS) 64
set ::env(GLB_RT_ADJUSTMENT) 0.15
run_routing
run_magic
```
```bash
# SPEF extraction
cd $WD/SPEF_Extractor
python3 main.py $RUN/tmp/merged.lef $RUN/results/routing/picorv32a.def

# View final layout
klayout $RUN/results/magic/picorv32a.gds
```
```tcl
# OpenSTA with parasitics
read_liberty $LIB_TYPICAL
read_verilog $NETLIST
link_design picorv32a
read_sdc my_base.sdc
read_spef picorv32a.spef
set_propagated_clock [all_clocks]
report_checks -path_delay max -format full_clock_expanded
report_checks -path_delay min -format full_clock_expanded
```
