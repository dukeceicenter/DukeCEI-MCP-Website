---
layout: default
title: Service API
parent: MCP-EDA

---

# Service API

---

## What the platform exposes

Seven small, single-purpose HTTP micro-services drive the end-to-end RTL-to-GDS flow. Each wraps a tool script (Synopsys Design Compiler or Cadence Innovus) and listens on its own port.

| Stage | Service Script(s) | Port(s) |
|:------|:------------------|:--------|
| **Synthesis** | `setup_server.py`, `compile_server.py` | 3333, 3334 |
| **Physical Implementation** | `floorplan_server.py`, `powerplan_server.py`, `placement_server.py`, `cts_server.py`, `route_server.py`, `save_server.py` | 3335 – 3340 |

## Common request fields

Every endpoint understands these three fields:

- **`design`** – logical design name (e.g. "des"). **Required**.
- **`tech`** – target technology directory under `scripts/`; default **FreePDK45**.
- **`force`** – `true` forces the stage to re-run even if artefacts already exist.

Each stage adds its own optional parameters, listed below. All parameters travel to the underlying TCL scripts via environment variables, so you can extend them freely without touching the Python wrappers.

## Endpoint responsibilities & accepted parameters

### `/setup/run` (3333) – "DC setup"
{: .no_toc }

Initialises Synopsys Design Compiler, loads libraries and applies constraints.

| Field | Type | Description |
|:------|:-----|:------------|
| `clock_period` | float | Target clock period in ns |
| `compile_effort` | string | Design Compiler compile mode ("compile" / "compile_ultra" / "incremental") |
| `lint_only` | bool | Run lint & constraint checks only, skip mapping |

### `/compile/run` (3334) – "DC compile"
{: .no_toc }

Maps RTL to gates and produces timing / area / power reports. Returns `syn_ver`, an auto-generated synthesis version string.

| Field | Type | Description |
|:------|:-----|:------------|
| `clock_period` | float | Overrides period from setup (optional) |
| `max_fanout` | int | Maximum net fan-out |
| `max_transition` | float | Maximum cell output transition (ns) |
| `power_effort` | string | low / medium / high |

### `/floorplan/run` (3335) – "Innovus floorplan"
{: .no_toc }

Creates die outline, IO ring and macro rows. Generates a unique `impl_ver` reused by later stages.

| Field | Type | Description |
|:------|:-----|:------------|
| `syn_ver` | string | Synthesis version coming from compile stage |
| `target_util` | float | Core utilisation (0-1) |
| `aspect_ratio` | float | Die aspect ratio (Y/X) |
| `flow_effort` | string | express / standard |

### `/power/run` (3336)
{: .no_toc }

Builds power grid (rings, stripes, vias).

| Field | Type | Description |
|:------|:-----|:------------|
| `impl_ver` | string | Implementation version from floorplan stage |
| `stripe_width` | float | Power stripe width (µm) |
| `stripe_pitch` | float | Centre-to-centre spacing (µm) |
| `metal_layers` | string | Comma-separated list e.g. "M4,M5" |

### `/place/run` (3337)
{: .no_toc }

Standard-cell placement plus pre-CTS optimisation.

| Field | Type | Description |
|:------|:-----|:------------|
| `impl_ver` | string | Implementation version |
| `timing_effort` | string | none / low / medium / high |
| `congestion_effort` | string | low / medium / high |
| `max_density` | float | Legaliser density threshold (0-1) |

### `/cts/run` (3338)
{: .no_toc }

Clock-tree synthesis and post-CTS optimisation.

| Field | Type | Description |
|:------|:-----|:------------|
| `impl_ver` | string | Implementation version |
| `buffer_cells` | string | Whitespace-separated list of clock buffer cells |
| `clock_gate_cells` | string | Whitespace-separated list of CG cells |
| `cell_density` | float | CTS cell density target |

### `/route/run` (3339)
{: .no_toc }

Global + detailed routing, then timing/DRC clean-up.

| Field | Type | Description |
|:------|:-----|:------------|
| `impl_ver` | string | Implementation version |
| `timing_driven` | bool | Enable timing-driven routing |
| `fix_clock_nets` | bool | Keep clock nets fixed |
| `si_aware` | bool | Enable SI-aware delay calculation |
| `drc_limit` | int | Stop verify_drc after this many violations |
| `top_module` | string | Override top verilog module (optional) |

### `/save/run` (3340)
{: .no_toc }

Writes `{impl_ver}.gds`, `.def`, `.sdf` to the implementation directory.

| Field | Type | Description |
|:------|:-----|:------------|
| `impl_ver` | string | Implementation version |
| `compress_gds` | bool | Gzip the GDS output |

## Response contract (all services)

Every endpoint returns the same JSON envelope:

```json
{
  "status"   : "ok" | "error: <msg>",
  "log_path" : "/abs/path/to/tool.log",
  "reports"  : {
       "some_report.rpt" : "full text …",
       "timing.rpt.gz"   : "…inflated text…"
  }
}
```

{: .note }
The HTTP status code is always **200** so that CI pipelines rely on the `status` field instead of transport exceptions.

## Typical end-to-end run

A helper script stitches the stages together:

```bash
./run_pipeline.sh \
  --design des \
  --tech   FreePDK45 \
  --clock_period 2.5   \   # any parameter accepted by /setup/run
  --target_util 0.65    \   # propagated down to floorplan
  --stripe_width 2.0    \   # picked up by /power/run
  --timing_driven true
```

## Troubleshooting quick tips

- **Directory not found** → stale `impl_ver`; rerun earlier stage or add `"force": true`.
- **Tool licence / crash** → open the file pointed to by `log_path`.
- **Malformed JSON** → FastAPI replies with 400.

## Contact

**Author:** Andy Lu • yl996@duke.edu  
**Repo:** [https://github.com/AndyLu666/MCP-EDA-Server](https://github.com/AndyLu666/MCP-EDA-Server)