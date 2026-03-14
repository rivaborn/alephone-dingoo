# Source_Files/RenderMain/OGL_Render.cpp

## File Purpose
Main OpenGL rendering interface for the Aleph One game engine (Marathon-based). Implements 3D scene rendering, projection matrix management, coordinate system transformations, UI rendering (crosshairs, text, HUD), and shader setup for 3D models and effects.

## Core Responsibilities
- OpenGL context lifecycle management (startup, shutdown, state initialization)
- Projection and view matrix setup and switching between 3D/2D rendering modes
- Coordinate transformation between Marathon world space, eye space, and OpenGL clip space
- 2D UI rendering: crosshairs, on-screen text, HUD elements
- 3D model rendering with multi-pass shader system (normal, glowing, static effects)
- Lighting and blending mode management
- Texture preloading to avoid runtime stalls
- Static/noise effect rendering using polygon stippling or stencil buffering
- Platform-specific rendering context handling (Mac AGL, Windows)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SurfaceCoords` | struct | Surface coordinate system with U/V texture vectors and orthogonal complements for ray-surface intersection |
| `ShaderDataStruct` | struct | Transient data passed to shader callbacks: model ptr, skin ptr, color, collection, CLUT |
| `LightingDataStruct` | struct | Lighting parameters for callback: light type, direction, directional + average color components, opacity |
| `ModelRenderer` | external class | 3D model renderer object managing shading pipeline |
| `ModelRenderShader` | external struct | Shader definition with texture/lighting callbacks and callback data pointers |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_OGL_IsActive` | bool | static | Whether OpenGL rendering is currently active |
| `JustInited` | bool | static | Flag indicating context just created; used to skip certain state updates |
| `ProjectionType` | int | static | Current projection mode (none/3D eye/2D screen); gates redundant matrix reloads |
| `Z_Buffering` | bool | static | Whether depth buffering is enabled |
| `CenteredWorld_2_MaraEye`, `World_2_MaraEye`, `World_2_OGLEye`, etc. | GLdouble[16] | static | Transformation matrices between coordinate systems |
| `XScale`, `YScale`, `XOffset`, `YOffset`, etc. | GLdouble | static | Screen-to-world conversion factors for ray casting |
| `HorizCoords`, `VertCoords` | `SurfaceCoords` | static | Surface coordinate systems for horizontal/vertical surfaces |
| `Yaw` | double | static | Current yaw angle (full circle = 1) |
| `LandscapeRescale` | double | static | Scaling factor for landscape rendering proximity |
| `SelfLuminosity` | _fixed | static | Miner's light effect intensity |
| `CurrFog`, `CurrFogColor` | `OGL_FogData*`, GLfloat[4] | static | Current fog state and color (may differ due to infravision) |
| `BlendType` | short | static | Last blend mode set; prevents redundant state changes |
| `StaticPatterns`, `StaticRandom` | GLuint[4][32], `GM_Random` | static | Stipple patterns and PRNG for static/noise effect |
| `ModelRenderObject`, `StandardShaders`, `StaticModeShaders` | `ModelRenderer`, `ModelRenderShader[]` | static | 3D model renderer and shader configurations |
| `ShaderData`, `LightingData` | `ShaderDataStruct`, `LightingDataStruct` | static | Transient shader callback data |

## Key Functions / Methods

### OGL_StartRun
- Signature: `bool OGL_StartRun(CGrafPtr WindowPtr)` (Mac) / `bool OGL_StartRun()` (other)
- Purpose: Initialize OpenGL rendering context and engine state for a game run
- Inputs: Window pointer (Mac only)
- Outputs/Return: true if successful, false if context creation failed
- Side effects: Creates GL context; loads all collections, textures, shaders; enables z-buffer, culling, alpha test, blending
- Calls: `aglChoosePixelFormat`, `aglCreateContext`, `aglSetDrawable`, `setup_gl_extensions`, `load_replacement_collections`, `OGL_StartTextures`, `FontSpecifier::OGL_ResetFonts`, `SetupShaders`, `PreloadTextures`
- Notes: Platform-specific (Mac vs Win32); respects graphics preferences for fullscreen, z-buffering, sRGB

### OGL_StopRun
- Signature: `bool OGL_StopRun()`
- Purpose: Teardown and destroy the OpenGL rendering context
- Outputs/Return: true if active context was destroyed, false if none was active
- Side effects: Stops texture management; destroys AGL context (Mac)
- Calls: `OGL_StopTextures`, `aglDestroyContext`

### SetProjectionType
- Signature: `static void SetProjectionType(int NewProjectionType)`
- Purpose: Switch projection matrix (3D perspective vs. 2D screen-aligned)
- Inputs: `Projection_OpenGL_Eye` or `Projection_Screen`
- Side effects: Loads appropriate matrix into `GL_PROJECTION`, updates static `ProjectionType`
- Notes: No-op if already in target projection; prevents redundant state changes

### OGL_RenderCrosshairs
- Signature: `bool OGL_RenderCrosshairs()`
- Purpose: Render player crosshair on screen
- Outputs/Return: true if rendered, false if OpenGL inactive
- Side effects: Sets 2D screen projection, disables textures, enables blending, draws crosshair quads/octagon in center of screen
- Calls: `GetCrosshairData`, `SetProjectionType`, `glBegin/End` with `GL_QUADS`/`GL_QUAD_STRIP`

### OGL_RenderText
- Signature: `bool OGL_RenderText(short BaseX, short BaseY, const char *Text, unsigned char r, unsigned char g, unsigned char b)`
- Purpose: Render a text string at screen coordinates with drop shadow
- Inputs: Base position (left, top), text string, RGB color
- Outputs/Return: true if rendered
- Side effects: Creates display list, draws black drop shadow offset, then foreground text in color
- Calls: `SetProjectionType`, `GetOnScreenFont`, `glGenLists`, `glNewList`, `glEndList`, `glCallList`, `glDeleteLists`

### LightingCallback
- Signature: `static void LightingCallback(void *Data, size_t NumVerts, GLfloat *Normals, GLfloat *Positions, GLfloat *Colors)`
- Purpose: Compute per-vertex lighting for 3D models (shader callback)
- Inputs: Lighting data struct, vertex normals, positions, output color array
- Side effects: Fills Colors array with per-vertex lighting based on normal/position and light direction
- Notes: Supports two modes (fast with/without fade); scales normal vectors by light matrix; handles semitransparent objects

### NormalShader, GlowingShader, StaticModeShader
- Signature: `void NormalShader(void *Data)`, `void GlowingShader(void *Data)`, `void StaticModeShader(void *Data)`
- Purpose: Texture shader callbacks for different rendering passes
- Side effects: **NormalShader**: sets color, loads normal skin texture, sets normal blend. **GlowingShader**: sets white color with alpha, enables blend, disables alpha test, loads glow texture. **StaticModeShader**: runs per-pass setup, loads silhouette or generates procedural noise texture
- Notes: Called by model renderer for each material; static mode uses 3-4 passes for RGB color channels

### OGL_Copy2D
- Signature: `bool OGL_Copy2D(GWorldPtr BufferPtr, Rect& SourceBounds, Rect& DestBounds, bool UseBackBuffer, bool FrameEnd)` (Mac only)
- Purpose: Copy 2D graphics (HUD, map, terminal) from off-screen buffer to display
- Inputs: Source GWorld, source/dest rectangles, buffer selection, frame-end flag
- Outputs/Return: true if copied
- Side effects: Converts pixel format (16/32-bit ARGB ΓåÆ RGBA 8888), reverses row order, draws via glRasterPos/glDrawPixels, optionally swaps buffers
- Notes: Uses strip buffer (`Buffer2D`) for efficient multi-line transfer; handles both 16-bit and 32-bit source formats

## Control Flow Notes
**Init**: `OGL_StartRun()` sets up context, collections, textures, shaders, and preloads critical textures.

**Per-frame**: Rendering calls set projection type dynamically, then render 3D geometry (handled elsewhere), 2D UI (`OGL_RenderCrosshairs`, `OGL_RenderText`, `OGL_Copy2D`).

**Shutdown**: `OGL_StopRun()` cleans up textures and context.

Model rendering uses a callback-based shader system; shaders are selected by `ModelRenderObject` and invoked for each pass/material, receiving transient data via `ShaderData` and `LightingData`.

## External Dependencies
- **Core**: `world.h` (coordinates), `map.h` (game world), `preferences.h`, `render.h`
- **OpenGL**: Platform headers (`<GL/gl.h>`, `<OpenGL/gl.h>`, or `<AGL/agl.h>`)
- **Engine subsystems**: `OGL_Textures.h`, `OGL_Faders.h`, `ModelRenderer.h`, `Crosshairs.h`, `Logging.h`
- **Symbols defined elsewhere**: `OGL_IsPresent()`, `Get_OGL_ConfigureData()`, `load_replacement_collections()`, `OGL_StartTextures()`, `SetInfravisionTint()`, `GetCrosshairData()`, `PreloadTextures()` (declared but not defined here)
