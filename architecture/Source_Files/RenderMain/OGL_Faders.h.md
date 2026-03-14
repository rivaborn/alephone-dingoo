# Source_Files/RenderMain/OGL_Faders.h

## File Purpose
Header file providing the OpenGL renderer interface for screen fading effects (color and transparency transitions). Manages a fader queue to support different types of fades (e.g., liquid, status effects) applied during rendering.

## Core Responsibilities
- Expose fader activity check and rendering entry point
- Define fader queue categories (Liquid vs. Other visual effects)
- Provide access to the fader queue for configuration
- Render accumulated faders to a screen region with color and alpha blending

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Fader` | struct | Encapsulates one fader: fade type, RGBA color channels, default constructor |
| `FaderQueue_*` | enum constants | Queue categories: `Liquid`, `Other`, `NUMBER_OF_FADER_QUEUE_ENTRIES` (count) |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_FaderActive
- Signature: `bool OGL_FaderActive()`
- Purpose: Query whether any active faders are queued for rendering this frame
- Inputs: None
- Outputs/Return: `bool` ΓÇö true if faders will be rendered
- Side effects: None (read-only query)
- Calls: None visible
- Notes: Guard for optimization; likely checked before calling `OGL_DoFades()`

### GetOGL_FaderQueueEntry
- Signature: `OGL_Fader *GetOGL_FaderQueueEntry(int Index)`
- Purpose: Access or configure a fader in the queue by index
- Inputs: `Index` ΓÇö queue position (0ΓÇôbased, likely one per `FaderQueue_*` type)
- Outputs/Return: Pointer to the `OGL_Fader` struct at that queue position
- Side effects: None apparent (accessor; actual modifications occur via returned pointer)
- Calls: None visible
- Notes: Allows caller to set `Type` and `Color[4]` before rendering

### OGL_DoFades
- Signature: `bool OGL_DoFades(float Left, float Top, float Right, float Bottom)`
- Purpose: Render all queued faders as overlays within the specified screen region
- Inputs: Screen-space bounding box (Left, Top, Right, Bottom in normalized/pixel coordinates)
- Outputs/Return: `bool` ΓÇö true if any faders were rendered
- Side effects: OpenGL state changes (blend mode, texture binding, etc.); framebuffer writes
- Calls: Not visible in this header (implementation in .cpp)
- Notes: Called once per frame during render pass; must occur after scene rendering, before swap

## Control Flow Notes
- Integrated into the frame/render pipeline; typically called after 3D scene and before UI
- `OGL_FaderActive()` acts as a guard to skip fader setup if queue is empty
- Queue is populated externally (likely by game logic setting `Type` and `Color` via `GetOGL_FaderQueueEntry()`)
- Faders persist across frames until explicitly cleared

## External Dependencies
- `config.h` ΓÇö guards entire interface with `HAVE_OPENGL` (disabled on platforms without OpenGL)
- Uses standard C types: `bool`, `short`, `float`, `int`
- Likely linked against OpenGL library (not visible in header)
