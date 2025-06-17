---
layout: default
title: Pipeline Overview
parent: MCP-EDA
---

# Pipeline Overview

The MCP-EDA Server implements a fully automated, end-to-end digital design flow by exposing each major EDA step as a lightweight HTTP microservice. Users simply clone the repository and invoke a single CLI script—run_pipeline.sh—to drive the complete flow, from RTL synthesis through final GDSII/SDF export.

---

![MCP-EDA Flow Diagram](/assets/images/flow.jpg){: .d-block .mx-auto }

## Quick Start

Get started with the MCP-EDA pipeline in just three commands:

```bash
git clone https://github.com/AndyLu666/MCP-EDA-Server.git
cd mcp-eda-example
./run_pipeline.sh
```

This script turns your shell (or CI job) into the orchestrator: it issues HTTP POST calls to each stage's server in turn.

## Synthesis Stage (Synopsys Design Compiler)

The synthesis stage transforms your RTL design into a gate-level netlist using Synopsys Design Compiler.

### Setup Server
{: .no_toc }

**Service:** `setup_server.py`  
**Port:** 3333

Loads the RTL, applies constraints via `frontend/1_setup.tcl`, and generates a mapped netlist.

### Compile Server
{: .no_toc }

**Service:** `compile_server.py`  
**Port:** 3334

Runs the actual compile and optimization commands defined in `frontend/2_synthesis.tcl`.

## Physical Implementation Stage (Cadence Innovus)

The physical implementation stage transforms the synthesized netlist into a complete physical layout using Cadence Innovus.

### Floorplan Server
{: .no_toc }

**Service:** `floorplan_server.py`  
**Port:** 3335

Defines die outlines, I/O pin assignments, and initial floorplan using:
- `backend/1_setup.tcl`
- `backend/2_floorplan.tcl`

### Power Plan Server
{: .no_toc }

**Service:** `powerplan_server.py`  
**Port:** 3336

Builds the on-chip power grid using `backend/3_powerplan.tcl`.

### Placement Server
{: .no_toc }

**Service:** `placement_server.py`  
**Port:** 3337

Performs timing-driven standard-cell placement using `backend/4_place.tcl`.

### CTS Server
{: .no_toc }

**Service:** `cts_server.py`  
**Port:** 3338

Synthesizes and balances the clock tree using `backend/5_cts.tcl`.

### Route Server
{: .no_toc }

**Service:** `route_server.py`  
**Port:** 3339

Carries out global and detailed routing with DRC checks using `backend/7_route.tcl`.

### Save Server
{: .no_toc }

**Service:** `save_server.py`  
**Port:** 3340

Finalizes the design, writing out final files using `backend/8_save_design.tcl`.

## Final Deliverables

At the end of the pipeline you automatically obtain:

| File | Description | Format |
|:-----|:------------|:-------|
| **`final.gds`** | GDSII layout | Binary layout format |
| **`final.def`** | DEF file | Design Exchange Format |
| **`final.sdf`** | SDF timing | Standard Delay Format |

{: .note }
> **Additional outputs** include detailed timing reports, DRC reports, power analysis, and area utilization summaries from each stage.
