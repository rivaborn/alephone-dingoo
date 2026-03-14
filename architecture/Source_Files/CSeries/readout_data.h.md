# Source_Files/CSeries/readout_data.h

## File Purpose
Declares global performance profiling structures and statistics counters for engine subsystems. Provides extern declarations for runtime metrics covering timing, rendering operations, and OpenGL texture management.

## Core Responsibilities
- Define timing statistics structure for profiling engine phases (world, AI, rendering, etc.)
- Declare pre-rendering statistics counters (nodes, windows, objects)
- Declare rendering statistics counters (surfaces, primitives, pixels)
- Declare OpenGL texture statistics and binding metrics
- Export global stat instances for engine-wide telemetry collection

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `time_usage_stats` | struct | Organizes Timer objects for profiling major engine phases: total, world (AI/collision), visibility, rendering (calc/obj/clip/draw), blitting |
| `prerender_stats` | struct | Counters for pre-pass node/window analysis (total, sorted, combined counts) |
| `render_stats` | struct | Counters for render output: vertical/horizontal/sprite surfaces with pixel counts and geometry stats |
| `OGL_TexturesStats` | struct | Tracks GPU texture cache state: in-use count, bind operation counts (total/min/max), and texture age metric |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gPrerenderStats` | `prerender_stats` | global | Collects pre-render pass statistics across frame |
| `gRenderStats` | `render_stats` | global | Accumulates rendering operation counts per frame |
| `gUsageTimers` | `time_usage_stats` | global | Tracks elapsed time in major engine subsystems |
| `gGLTxStats` | `OGL_TexturesStats` | global | Monitors GPU texture state and bind performance |

## Key Functions / Methods

### OGL_TexturesStats (constructor)
- **Signature:** `OGL_TexturesStats()`
- **Purpose:** Initialize texture statistics counters to safe defaults
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Zeroes `inUse`, `binds`, `totalBind`; sets `minBind=500000`; zeroes remaining fields
- **Calls:** None
- **Notes:** Constructor assumes 500,000 is a safe upper bound for minimum bind count; resets are needed each measurement cycle

## Control Flow Notes
This module fits into frame profiling/instrumentation. Statistics are populated during frame execution (world simulation ΓåÆ visibility ΓåÆ rendering pipeline) and likely read by a HUD display or log output. The `Timer` class (from included `timer.h`) drives time-based profiling; counters are incremented throughout rendering and pre-rendering passes.

## External Dependencies
- **Includes:** `timer.h` ΓÇö provides `Timer` class for high-resolution timing
- **Extern definitions:** All four global stat variables are declared `extern` here but defined in another translation unit (likely engine core or main application file)
