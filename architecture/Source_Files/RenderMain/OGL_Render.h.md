# Source_Files/RenderMain/OGL_Render.h

## File Purpose
Public interface for OpenGL rendering functionality in the Marathon/Aleph One game engine. Declares the API for screen management, view setup, object rendering, and 2D graphics integration. Separates rendering calls from configuration/parameter access (handled by OGL_Setup.h).

## Core Responsibilities
- Screen initialization, teardown, and buffer management
- View transformation and perspective configuration
- Rendering of 3D geometry (walls, sprites) and UI elements (crosshairs, text)
- Infravision tinting effects
- Integration of 2D graphics (status bar, map, terminal) with OpenGL
- Window bounds and rendering target management
- Foreground/background rendering modes (e.g., weapons in hand)

## Key Types / Data Structures
None defined in this file. External types used:
| Name | Kind | Purpose |
|------|------|---------|
| `polygon_definition` | struct (undefined) | Describes wall geometry for rendering |
| `rectangle_definition` | struct (undefined) | Describes sprite/quad geometry for rendering |
| `view_data` | struct (undefined) | Camera/viewport parameters for perspective setup |
| `Rect` | struct (undefined) | Screen/window bounds |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_IsActive
- Signature: `bool OGL_IsActive()`
- Purpose: Test whether OpenGL is currently active for rendering
- Inputs: None
- Outputs/Return: `true` if OpenGL is active; `false` otherwise
- Side effects: None
- Calls: None visible
- Notes: Returns `false` if OpenGL is not present on the system

### OGL_ClearScreen
- Signature: `bool OGL_ClearScreen()`
- Purpose: Clear the screen to black and prepare for rendering
- Inputs: None
- Outputs/Return: `true` on success
- Side effects: Clears the framebuffer
- Calls: None visible
- Notes: Gated by `HAVE_OPENGL` preprocessor flag

### OGL_StartRun / OGL_StopRun
- Signature: `bool OGL_StartRun()` (or with `CGrafPtr WindowPtr` on Mac); `bool OGL_StopRun()`
- Purpose: Create/destroy the OpenGL rendering context
- Inputs: (Mac only) GrafPort for the window
- Outputs/Return: `true` on success
- Side effects: Allocates/deallocates rendering context and associated resources
- Calls: None visible
- Notes: Platform-specific; Mac version takes a window pointer

### OGL_SetWindow
- Signature: `bool OGL_SetWindow(Rect &ScreenBounds, Rect &ViewBounds, bool UseBackBuffer)`
- Purpose: Configure rendering window bounds and back buffer usage
- Inputs: Screen boundaries, view boundaries, flag for back buffer
- Outputs/Return: `true` on success
- Side effects: Modifies OpenGL viewport and buffer configuration
- Calls: None visible
- Notes: Called when window or viewport changes

### OGL_SetView
- Signature: `bool OGL_SetView(view_data &View)`
- Purpose: Configure camera/perspective for main 3D rendering
- Inputs: View data structure (position, orientation, FOV)
- Outputs/Return: `true` on success
- Side effects: Updates OpenGL projection and model-view matrices
- Calls: None visible
- Notes: Sets up proper perspective rendering; controls back buffer usage

### OGL_SetInfravisionTint
- Signature: `bool OGL_SetInfravisionTint(short Collection, bool IsTinted, float Red, float Green, float Blue)`
- Purpose: Configure infravision color overlay for a texture collection
- Inputs: Collection ID, tint enable flag, RGB values (0ΓÇô1 range)
- Outputs/Return: `true` on success
- Side effects: Modifies rendering state for subsequent texture rendering
- Calls: None visible
- Notes: Used for special vision modes in gameplay

### OGL_RenderWall / OGL_RenderSprite
- Signature: `bool OGL_RenderWall(polygon_definition& RenderPolygon, bool IsVertical)` / `bool OGL_RenderSprite(rectangle_definition& RenderRectangle)`
- Purpose: Render wall geometry or sprite/billboard quad
- Inputs: Polygon/rectangle definition, (wall only) vertical flag
- Outputs/Return: `true` on success
- Side effects: Writes to framebuffer
- Calls: None visible
- Notes: Core geometry rendering for 3D world

### OGL_RenderCrosshairs / OGL_RenderText
- Signature: `bool OGL_RenderCrosshairs()` / `bool OGL_RenderText(short BaseX, short BaseY, const char *Text, unsigned char r, g, b)`
- Purpose: Render HUD crosshairs or text overlay
- Inputs: (Text only) screen position, C-string, RGB color
- Outputs/Return: `true` on success
- Side effects: Writes to framebuffer
- Calls: None visible
- Notes: Text rendering uses optional color parameters (default white)

### OGL_StartMain / OGL_EndMain
- Signature: `bool OGL_StartMain()` / `bool OGL_EndMain()`
- Purpose: Delimit main view rendering section
- Inputs: None
- Outputs/Return: `true` on success
- Side effects: Pushes/pops OpenGL state, manages render passes
- Calls: None visible
- Notes: Bracket all primary 3D geometry rendering

### OGL_SwapBuffers
- Signature: `bool OGL_SwapBuffers()`
- Purpose: Present the rendered frame to the display
- Inputs: None
- Outputs/Return: `true` on success
- Side effects: Swaps back and front buffers; blocks until vsync if enabled
- Calls: None visible
- Notes: Called once per frame after all rendering

### OGL_Get2D
- Signature: `bool OGL_Get2D()`
- Purpose: Query whether 2D graphics should be rendered via OpenGL
- Inputs: None
- Outputs/Return: `true` if 2D should use OpenGL; `false` for fallback
- Side effects: None
- Calls: None visible

### OGL_Copy2D
- Signature: `bool OGL_Copy2D(GWorldPtr BufferPtr, Rect& SourceBounds, Rect& DestBounds, bool UseBackBuffer, bool FrameEnd)` (Mac only)
- Purpose: Copy 2D graphics (HUD, map, terminal) into the framebuffer
- Inputs: Source GWorld, source/dest rectangles, back buffer flag, frame-end flag
- Outputs/Return: `true` on success
- Side effects: Copies pixel data into framebuffer; may finalize frame
- Calls: None visible
- Notes: Mac-specific; integrates CPU-rendered 2D with OpenGL 3D

### OGL_SetForeground / OGL_SetForegroundView
- Signature: `bool OGL_SetForeground()` / `bool OGL_SetForegroundView(bool HorizReflect)`
- Purpose: Configure rendering mode for foreground objects (weapons in hand)
- Inputs: (SetForegroundView only) horizontal reflection flag
- Outputs/Return: `true` on success
- Side effects: Updates projection/view matrix for weapon rendering
- Calls: None visible
- Notes: Special orthographic or adjusted perspective for HUD-like objects

## Control Flow Notes
This file defines the frame-level rendering API. Typical frame sequence:
1. **Init**: `OGL_StartRun()` once at startup
2. **Per-frame**:
   - `OGL_SetView()` with camera data
   - `OGL_StartMain()` + `OGL_RenderWall()` + `OGL_RenderSprite()` + `OGL_EndMain()` for main scene
   - `OGL_SetForeground()` + weapon rendering for HUD weapons
   - `OGL_Copy2D()` for status bar/map/terminal (Mac only)
   - `OGL_RenderCrosshairs()` and `OGL_RenderText()` for UI overlays
   - `OGL_SwapBuffers()` to present
3. **Shutdown**: `OGL_StopRun()` once at exit

## External Dependencies
- **Includes**: `OGL_Setup.h` (configuration structures and helper functions), `config.h` (HAVE_OPENGL feature gate)
- **Undefined types** (defined elsewhere): `polygon_definition`, `rectangle_definition`, `view_data`, `Rect`, `GWorldPtr`, `CGrafPtr`, `RGBColor`
- **Preprocessing**: Gated by `HAVE_OPENGL` macro; some functions Mac-only (`#ifdef mac`)
