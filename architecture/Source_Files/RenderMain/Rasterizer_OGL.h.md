# Source_Files/RenderMain/Rasterizer_OGL.h

## File Purpose
OpenGL-based implementation of the Rasterizer interface. This thin wrapper class delegates to C-style OpenGL rendering functions from OGL_Render.h, providing an object-oriented API for the game engine's rendering pipeline.

## Core Responsibilities
- Implement the RasterizerClass virtual interface for OpenGL rendering
- Configure view and camera parameters before frame rendering
- Route polygon and sprite rendering to underlying OGL_Render functions
- Manage foreground vs. background rendering modes (weapons, UI layers)
- Delegate actual GPU commands to C-style OGL functions

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Methods

### SetView
- **Signature:** `void SetView(view_data& View)`
- **Purpose:** Configure the rasterizer's camera/view parameters before rendering
- **Inputs:** `view_data&` ΓÇô view configuration (camera position, frustum, etc.)
- **Outputs/Return:** void
- **Side effects:** Modifies OpenGL state via OGL_SetView()
- **Calls:** `OGL_SetView()`
- **Notes:** Must be called before Begin() or any rendering calls

### SetForeground
- **Signature:** `virtual void SetForeground()`
- **Purpose:** Switch to foreground rendering mode (weapons, HUD overlays)
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Modifies OpenGL state via OGL_SetForeground()
- **Calls:** `OGL_SetForeground()`

### SetForegroundView
- **Signature:** `virtual void SetForegroundView(bool HorizReflect)`
- **Purpose:** Configure view transformation for a foreground object with optional horizontal reflection
- **Inputs:** `bool HorizReflect` ΓÇô whether to mirror the view horizontally
- **Outputs/Return:** void
- **Side effects:** Modifies OpenGL matrix state via OGL_SetForegroundView()
- **Calls:** `OGL_SetForegroundView()`

### Begin / End
- **Purpose:** Frame lifecycle markersΓÇöBegin() starts frame rendering, End() finalizes it
- **Calls:** `OGL_StartMain()` / `OGL_EndMain()`

### texture_horizontal_polygon / texture_vertical_polygon / texture_rectangle
- **Purpose:** Render wall polygons (horizontal/vertical distinction suggests floor/ceiling vs. walls) and sprites
- **Calls:** `OGL_RenderWall()` with orientation flag; `OGL_RenderSprite()`
- **Notes:** Orientation parameter (true/false) differentiates vertical vs. horizontal wall geometry

## Control Flow Notes
Typical render frame:
1. SetView() ΓÇô configure camera
2. Begin() ΓÇô start frame
3. Optionally: SetForeground() + SetForegroundView() ΓÇô prepare weapon/UI layer
4. Multiple texture_*() calls ΓÇô render scene geometry and sprites
5. End() ΓÇô finalize frame

## External Dependencies
- **Rasterizer.h** ΓÇô base class RasterizerClass
- **config.h** ΓÇô `HAVE_OPENGL` preprocessor guard
- **OGL_Render.h** (implied) ΓÇô declares OGL_SetView, OGL_StartMain, OGL_EndMain, OGL_SetForeground, OGL_SetForegroundView, OGL_RenderWall, OGL_RenderSprite (defined elsewhere)
- **render.h** (via Rasterizer.h) ΓÇô likely defines polygon_definition, rectangle_definition, view_data types
