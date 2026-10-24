# OpenLane Sky130 Physical Design Workshop

A 5-day hands-on workshop on the RTL → GDSII flow using open-source EDA tools and the SkyWater 130nm (Sky130) PDK, with the picorv32a RISC-V core as the example design.

## Day 1 – Inception of Open-Source EDA, OpenLANE and Sky130 PDK

Introduces how software reaches hardware (application → compiler → assembler → RISC-V binary → gates), the package/chip/pad/core/die anatomy, and the role of the PDK. Covers the full ASIC flow (synthesis, floorplan, placement, CTS, routing, signoff) and the OpenLANE tool chain. Lab: launch OpenLANE interactively, prepare picorv32a, run synthesis and compute the flop ratio.

## Day 2 – Good Floorplan vs Bad Floorplan & Library Cells

Explains floorplanning decisions: utilization factor, aspect ratio, pre-placed cells, decoupling capacitors, power planning and pin placement. Also covers library binding and placement, and how standard cells are characterized (delay and slew definitions). Lab: run floorplan and placement, then inspect the layouts in Magic.

## Day 3 – Custom Library Cell Design using Magic & ngspice

Walks through designing a standard cell inverter: layout rules, DRC fixing in Magic, parasitic R/C extraction to a SPICE netlist, and transient simulation in ngspice to measure propagation delay and transition time. Finishes by exporting the cell's LEF for use in the flow.

## Day 4 – Pre-Layout Timing Analysis & Clock Tree Synthesis

Covers delay lookup tables (slew × load), setup and hold analysis with ideal clocks in OpenSTA, and why a good clock tree matters. Introduces TritonCTS (H-tree, clock buffering, crosstalk and shielding). Lab: plug the custom cell into the flow, run CTS, and re-analyze timing with real clocks.

## Day 5 – Final Steps for RTL2GDS using TritonRoute & OpenSTA

Covers global routing (FastRoute) and detailed routing (TritonRoute) under design rules, power distribution network generation, SPEF parasitic extraction and post-route STA. Ends with GDSII generation for fabrication and viewing the final layout in KLayout.
