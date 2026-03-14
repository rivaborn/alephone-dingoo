# Source_Files/RenderMain/Rasterizer.h

## File Purpose
Abstract base class defining the interface for rasterizer implementations in the Aleph One game engine. Provides virtual methods for view setup, foreground object rendering, and texture rasterization that subclasses (e.g., software or OpenGL renderers) override with backend-specific implementations.

## Core Responsibilities
- Define the contract for rasterizer backend implementations
- Manage view state configuration via `SetView()`
- Handle foreground rendering setup (weapons in hand, HUD objects)
- Provide texture rendering entry points for polygons and rectangles
- Support frame lifecycle with `Begin()` and `End()` methods
- Enable horizontal reflection of foreground objects

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `RasterizerClass` | class | Abstract base class; subclassed by concrete rasterizer implementations |

## Global / File-Static State
None.

## Key Functions / Methods

### SetView
- Signature: `virtual void SetView(view_data& View)`
- Purpose: Configure the rasterizer with the current view/camera parameters
- Inputs: Reference to `view_data` struct containing camera position, FOV, screen dimensions, etc.
- Outputs/Return: None
- Side effects: Updates internal rendering state (assumed by subclasses)
- Calls: None (interface method)
- Notes: Must be called before any rendering; empty in base class

### SetForeground
- Signature: `virtual void SetForeground()`
- Purpose: Switch rasterizer mode to render foreground objects (weapons in hand)
- Inputs: None
- Outputs/Return: None
- Side effects: Changes rendering context/state for HUD/weapon layer
- Calls: None
- Notes: Typically called between main scene rendering and HUD rendering

### SetForegroundView
- Signature: `virtual void SetForegroundView(bool HorizReflect)`
- Purpose: Configure foreground object view with optional horizontal mirroring
- Inputs: `HorizReflect` ΓÇô whether to mirror the foreground object horizontally
- Outputs/Return: None
- Side effects: Updates foreground rendering parameters
- Calls: None

### Begin / End
- Signature: `virtual void Begin()` and `virtual void End()`
- Purpose: Bracket rendering operations; `Begin()` prepares for frame rendering, `End()` finalizes
- Inputs: None
- Outputs/Return: None
- Side effects: Frame setup/teardown (backend-specific)
- Calls: None

### Texture Rendering Methods
Three methods for rasterizing textured geometry:
- `texture_horizontal_polygon(polygon_definition& textured_polygon)`
- `texture_vertical_polygon(polygon_definition& textured_polygon)`
- `texture_rectangle(rectangle_definition& textured_rectangle)`

All share: **Inputs:** geometry with texture data; **Outputs:** None; **Side effects:** Submit geometry to rendering backend; **Calls:** None (interface).

## Control Flow Notes
This class sits at the rendering-backend abstraction boundary. It is called during the **render phase** of the main loop: after visibility/clipping (handled elsewhere), `Begin()` starts a frame, view is configured via `SetView()`, foreground/main geometry is textured via the polygon/rectangle methods, and `End()` completes the frame. Subclasses implement these methods for their specific backend (software rasterizer, OpenGL, etc.).

## External Dependencies
- `#include "render.h"` ΓÇô provides `view_data`, `polygon_definition`, `rectangle_definition` types and rendering constants
- `#ifdef HAVE_OPENGL` with `#include "OGL_Render.h"` ΓÇô optional OpenGL backend support (conditional compilation)
- Dependency on types defined in bundled headers (`render.h`, optionally `OGL_Render.h`)
