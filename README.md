# soc-design-and-planning-vsd
RTL-to-GDSII physical design flow using OpenLANE & Sky130 PDK | VSD SoC Design and Planning Workshop.

# Digital VLSI SoC Design and Planning — RTL to GDSII
A 2-week hands-on workshop on complete RTL-to-GDSII flow for digital VLSI SoC design, organised by VSD (VLSI System Design) in collaboration with NASSCOM. This repository documents my learning, lab outputs, and key takeaways from each day.

# Day 1 — Inception of Open-Source EDA, OpenLANE & Sky130 PDK
**Understanding the Chip Package**

When we look at any embedded board and point to what we call the "chip," we're actually looking at the package — a protective casing around the actual silicon die. The real chip sits in the centre of this package and communicates with the outside world via wire bonding — tiny wires that connect the chip's pads to the package pins.

**Inside the Chip: Core, Pads, and Die**

Zooming into the chip itself, all signals between the chip and the external world pass through pads placed around the periphery. The region enclosed by the pads is called the core — this is where all the actual digital logic lives. Together, the core and the pads form the die, which is the fundamental unit of chip manufacturing.

1) **Foundry** — the place where chips are physically manufactured
2) **Foundry IPs** — IP blocks that require specialized process knowledge to implement (e.g., PLLs, SRAMs)
3) **Macros** — reusable, purely digital logic blocks
   
**From Software to Silicon — The ISA Bridge**
A C program running on a chip goes through a multi-layer transformation:

1) The C code is compiled into RISC-V assembly (or another ISA)
2) The assembler converts it to binary machine code (0s and 1s)
3) This binary pattern needs an RTL implementation of the ISA
4) The RTL gets synthesized and goes through the full PnR (Place and Route) flow to become a physical layout
   
The system software stack (OS → Compiler → Assembler) acts as the bridge between what the programmer writes and what the hardware executes.

**Why Open-Source EDA Matters**

For a fully open-source ASIC design flow, three things are needed:

1. **RTL Designs** (e.g., from opencores.org)
2. **EDA Tools** (synthesis, P&R, verification)
3. **PDK Data** (process-specific design rules, standard cell libraries)
   
Historically, PDKs were proprietary and distributed only under NDAs, making chip design inaccessible to most people. This changed in June 2020, when Google collaborated with SkyWater Technology to release the Sky130 PDK as the world's first open-source process design kit — a massive milestone for the VLSI community.

OpenLANE and the Automated RTL to GDSII Flow
OpenLANE is an open-source flow built on top of multiple EDA tools that automates the journey from an RTL netlist all the way to the final GDSII layout file. It uses:

| Stage           | Tool(s) Used           |
| --------------- | ---------------------- |
| Synthesis       | Yosys, ABC             |
| Floorplan & PDN | OpenROAD               |
| Placement       | OpenROAD               |
| CTS             | TritonCTS              |
| Routing         | FastRoute, TritonRoute |
| SPEF Extraction | OpenRCX                |
| GDS Streaming   | Magic, KLayout         |
| Timing Analysis | OpenSTA                |

# Lab — Running OpenLANE for SPM

**Setting Up and Invoking OpenLANE**

The very first step is to navigate to the OpenLANE working directory and launch the tool in interactive mode, which lets us run each stage step-by-step.

```bash
cd /home/vscode/Desktop/OpenLane
make mount
./flow.tcl -interactive
package require openlane 1.0.2
```

<img width="823" height="594" alt="Screenshot (552)" src="https://github.com/user-attachments/assets/a691d07b-0a6b-4233-9896-8cf751e79135" />

 **Preparing the Design**

Before running synthesis, we prepare the design to merge the cell LEF and technology LEF files, and set up the run directory.

prep -design spm

<img width="1366" height="768" alt="Screenshot (564)" src="https://github.com/user-attachments/assets/819a903e-cf1e-4525-8fe7-afec4d7a52da" />

**Running Synthesis**

run_synthesis

<img width="1366" height="768" alt="Screenshot (569)" src="https://github.com/user-attachments/assets/a9613633-e315-4cb5-a14b-7a69b11b34c0" />

After synthesis completes, we can calculate the flop ratio — a useful sanity check:

```text
Flop Ratio = (No. of D Flip-Flops) / (Total No. of Cells)
           = 64 / 301
           ≈ 0.2126 → ~21.26%
```

# Day 2 — Floorplanning and Introduction to Library Cells
Chip Floorplanning — Core Area and Utilisation
Floorplanning is about deciding where everything goes on the chip. Two key parameters drive this:

1. Utilisation Factor = (Area occupied by Netlist) / (Total Core Area)
  A utilisation of 0.5–0.6 is typical — you want room for buffers, routing, etc.
2. Aspect Ratio = Height / Width of the core
 A ratio of 1 means a square; anything else is a rectangle.

**Pre-Placed Cells and Decoupling Capacitors**

**Pre-placed cells** (like memories, PLLs, and complex IP blocks) are fixed in position before automated placement runs. Their location is determined manually based on connectivity and power intent.

**Decoupling capacitors** are placed around pre-placed cells to act as local charge reservoirs — they compensate for voltage drops caused by switching activity and ensure these blocks see clean power.

**Power Planning — Mesh vs Ring**

A good power grid uses both power rings around the core and a power mesh across the chip. Multiple VDD and VSS rails are distributed in both metal layers so that every standard cell has a nearby power tap, minimising IR drop and electromigration risk.

**Pin Placement and Logical Cell Blockage**

Input and output pins are placed along the chip boundary. The relative placement of pins is guided by connectivity — a pin that drives logic deep in the core should be closer to that logic. The area between the core and the die boundary (I/O ring area) is blocked from automated cell placement to reserve it for pin buffers and ESD cells.

# Lab — Floorplan and Placement

Running Floorplan

run_floorplan

<img width="1366" height="768" alt="Screenshot (578)" src="https://github.com/user-attachments/assets/820d74d9-afb4-44ef-857c-350e7578180d" />

After this completes, we can inspect the DEF file that was generated:
```text
cd results/floorplan/
less spm.def
```

Viewing the Floorplan in Magic

```text
magic -T /home/vsduser/Desktop/OpenLane/designs/picorv32a/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.nom.lef \
      def read spm.def &
```
<img width="1366" height="768" alt="Screenshot (576)" src="https://github.com/user-attachments/assets/3c12be74-4eb6-40fb-b00c-87eb2074e209" />

<img width="1366" height="768" alt="Screenshot (630)" src="https://github.com/user-attachments/assets/6cbc2f8d-5c0a-4084-a778-aae68427e2dc" />

<img width="1366" height="768" alt="Screenshot (631)" src="https://github.com/user-attachments/assets/488bb5be-dd9c-4606-8e30-bc810d5d2960" />

<img width="1366" height="768" alt="Screenshot (634)" src="https://github.com/user-attachments/assets/91cd9408-b776-4829-a296-1399c3f9c799" />

<img width="1366" height="768" alt="Screenshot (635)" src="https://github.com/user-attachments/assets/1a5b1f61-8ed6-4b2e-ad2d-f83fb2e1fd7e" />

```text
run_placement
```
<img width="742" height="384" alt="Screenshot (736)" src="https://github.com/user-attachments/assets/c0351c3a-5c4a-4897-b30e-a6f2323e0d11" />













