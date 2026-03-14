# Source_Files/RenderOther/OGL_Blitter.cpp

## File Purpose
OpenGL implementation of a 2D image blitter for rendering SDL surfaces as textured quads. Handles surface-to-texture conversion, tiling for size limits, and rendering with scaling, rotation, and color tinting. Part of the Aleph One engine's 2D rendering pipeline.

## Core Responsibilities
- Load SDL surfaces into OpenGL textures, applying power-of-two padding and edge smearing
- Tile large surfaces across multiple textures (max 256├ù256) to respect hardware limits
- Render textured quads to screen with transformations (scale, rotate, tint, crop)
- Manage texture lifecycle and lazy-load on first draw
- Track active blitters via static registry for coordinated cleanup
- Set up orthographic projection matrix for 2D rendering over 3D scene

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Blitter` | class | Main image blitter; inherits from `Image_Blitter` |
| `SDL_Rect` | struct (external) | Source/destination rectangles and tile bounds |
| `SDL_Surface` | struct (external) | Source image data (inherited from parent) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_blitter_registry` | `set<OGL_Blitter*>` | static member | Tracks all active blitters for coordinated unload (e.g., on context loss) |
| `tile_size` | `const int` | static member | Max texture dimension (256); power-of-two for OpenGL compatibility |

## Key Functions / Methods

### OGL_Blitter()
- **Signature:** `OGL_Blitter()`
- **Purpose:** Initialize blitter with empty source/dest rectangles and unloaded state.
- **Side effects:** Sets `m_textures_loaded = false`

### _LoadTextures()
- **Signature:** `void _LoadTextures()`
- **Purpose:** Convert SDL surface into tiled OpenGL textures, handling power-of-two padding and edge clamping.
- **Inputs:** Uses member `m_surface` (source image) and `m_src` (source region)
- **Outputs/Return:** Populates `m_rects` (tile boundaries), `m_refs` (OpenGL texture IDs), `m_tile_width`/`m_tile_height`
- **Side effects:** Allocates OpenGL textures via `glGenTextures()`; registers this blitter in `m_blitter_registry`; sets `m_textures_loaded = true`
- **Calls:** `NextPowerOfTwo()` (defined in OGL_Textures.cpp), `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, OpenGL calls (`glGenTextures()`, `glTexImage2D()`, etc.)
- **Notes:** Smears right/bottom edge pixels to prevent texture boundary artifacts when using `GL_CLAMP_TO_EDGE`. Respects `OGL_Flag_TextureFix` config flag to enforce minimum tile size (128├ù128) on older Apple GPUs.

### _UnloadTextures()
- **Signature:** `void _UnloadTextures()`
- **Purpose:** Delete OpenGL textures and clear tile metadata.
- **Side effects:** Calls `glDeleteTextures()`; deregisters from `m_blitter_registry`; clears `m_refs`, `m_rects`; sets `m_textures_loaded = false`
- **Calls:** `glDeleteTextures()`

### Unload()
- **Signature:** `void Unload()`
- **Purpose:** Public cleanup entry point; calls parent class cleanup then texture unload.
- **Calls:** `Image_Blitter::Unload()`, `_UnloadTextures()`

### Draw() (overloads)
- **Signature:** `void Draw(const SDL_Rect& dst, const SDL_Rect& src)`
- **Purpose:** Public interface; rescales source rect if `m_scaled_src` differs from `m_src` (e.g., for UI scaling), then delegates to `_Draw()`.
- **Inputs:** `dst` (screen position/size), `src` (image region to render)
- **Calls:** `_Draw()`

### _Draw()
- **Signature:** `void _Draw(const SDL_Rect& dst, const SDL_Rect& src)`
- **Purpose:** Core rendering: bind textures, set up OpenGL state, iterate tiles, compute UV/vertex coords, emit textured quads.
- **Inputs:** `dst` (screen rect), `src` (source rect within loaded surface)
- **Side effects:** Sets OpenGL matrix mode, projection, blend func, and vertex data; renders quads via `glBegin(GL_QUADS)`/`glEnd()`
- **Calls:** `_LoadTextures()` (lazy load), OpenGL state/rendering calls
- **Notes:** Supports rotation via matrix transforms (translate ΓåÆ rotate ΓåÆ translate back). Applies tint color (`tint_color_r/g/b/a`). Skips tiles outside the source rect for efficiency. Disables sRGB if active to preserve color fidelity. Uses `glPushAttrib()/glPopAttrib()` to isolate GL state.

### BoundScreen()
- **Signature:** `static void BoundScreen()`
- **Purpose:** Set up 2D orthographic projection matching screen dimensions.
- **Side effects:** Calls `glMatrixMode()`, `glLoadIdentity()`, `glViewport()`, `glOrtho()` to map screen coordinates to GL clip space
- **Calls:** `glMatrixMode()`, `glLoadIdentity()`, `glViewport()`, `glOrtho()`, `ScreenWidth()`, `ScreenHeight()`

### ScreenWidth() / ScreenHeight()
- **Signature:** `static int ScreenWidth()` / `static int ScreenHeight()`
- **Purpose:** Query active SDL video surface dimensions.
- **Outputs/Return:** Width/height in pixels
- **Calls:** `SDL_GetVideoSurface()`

### StopTextures()
- **Signature:** `static void StopTextures()`
- **Purpose:** Emergency unload of all textures in registry (used on context loss or shutdown).
- **Side effects:** Iterates registry and calls `_UnloadTextures()` on each; registry self-modifies during iteration
- **Notes:** Pattern: erases current iterator and restarts from begin; ensures cleanup even if unload modifies registry

### ~OGL_Blitter()
- **Signature:** `~OGL_Blitter()`
- **Purpose:** Destructor; calls `Unload()`

## Control Flow Notes
This blitter integrates into a 2D-over-3D rendering model:
1. **Setup:** `BoundScreen()` called once per frame to set orthographic matrix
2. **Rendering:** `Draw()` invoked for UI/HUD elements; lazily loads textures on first call
3. **Teardown:** `StopTextures()` called on video context loss (e.g., mode change, shutdown)

The static registry pattern suggests this is used for coordinated resource management across multiple blitter instances during engine lifecycle events.

## External Dependencies
- **SDL:** `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`, `SDL_SetAlpha()`, `SDL_GetVideoSurface()`, `SDL_Rect`, `SDL_Surface`
- **OpenGL:** `glGenTextures()`, `glBindTexture()`, `glTexParameteri()`, `glTexImage2D()`, `glDeleteTextures()`, matrix/viewport/blend calls
- **Engine:** `Image_Blitter` (parent class, defined elsewhere), `OGL_Setup.h` (config access via `Get_OGL_ConfigureData()`, sRGB global `Using_sRGB`), `OGL_Textures.cpp` (`NextPowerOfTwo()`)
- **Standard:** `<vector>`, `<set>`, `<cmath>` (implicit via includes)
