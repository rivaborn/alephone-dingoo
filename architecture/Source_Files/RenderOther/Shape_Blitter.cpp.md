# Source_Files/RenderOther/Shape_Blitter.cpp

## File Purpose
Renders 2D UI bitmaps from the game's shape resources to screen. Supports dual rendering backends: OpenGL (hardware-accelerated) and SDL software rasterization. Handles shape scaling, rotation, cropping, and tinting.

## Core Responsibilities
- Construct Shape_Blitter objects from collection/texture descriptors
- Lazy-load and cache SDL surfaces for shapes
- Rescale shapes with proportional crop-rectangle adjustment
- Render shapes via OpenGL with texture manager setup and multiple texture-type handling
- Render shapes via SDL with surface transformations (rotation, flip, rescale)
- Apply visual effects: tinting (RGBA), rotation about center, cropping
- Manage surface memory lifecycle (allocation, caching, deallocation)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Shape_Blitter | class | Encapsulates shape rendering state and dual-backend rendering logic |
| TextureManager | class (external) | Manages OpenGL texture setup, color tables, and rendering parameters |
| SDL_Rect | struct (external) | Represents rectangle bounds for source/dest/crop regions |
| SDL_Surface | struct (external) | SDL surface pixel data and format metadata |

## Global / File-Static State
None.

## Key Functions / Methods

### Shape_Blitter (Constructor)
- Signature: `Shape_Blitter(short collection, short texture_index, short texture_type, short clut_index = 0)`
- Purpose: Initialize a shape blitter for a specific texture, populate dimensions from the shapes file.
- Inputs: Collection index, texture/shape index, texture type enum, optional color-lookup-table index
- Outputs/Return: Constructs object with m_desc, m_surface/m_scaled_surface pointers, dimension rects
- Side effects: Calls `get_shape_surface()` to load shape from resource; may allocate temporary SDL_Surface and pixel buffer
- Calls: `BUILD_DESCRIPTOR`, `BUILD_COLLECTION`, `get_shape_surface`, `SDL_FreeSurface`, `free`
- Notes: Initializes crop_rect to full shape bounds; surfaces are NULL until first use of SDL_Draw

### Rescale
- Signature: `void Rescale(int width, int height)`
- Purpose: Adjust displayed size and scale crop rectangle proportionally.
- Inputs: New width and height
- Outputs/Return: None (modifies m_scaled_src and crop_rect in-place)
- Side effects: None (metadata only)
- Calls: None
- Notes: Recomputes crop_rect x/y/w/h by scaling proportions; no memory allocation/deallocation

### Width / Height / UnscaledWidth / UnscaledHeight
- Signature: `int Width()`, `int Height()`, `int UnscaledWidth()`, `int UnscaledHeight()`
- Purpose: Query current and original shape dimensions.
- Inputs: None
- Outputs/Return: Dimension in pixels
- Side effects: None
- Calls: None
- Notes: Trivial accessors; width/height return scaled dimensions, unscaled variants return original

### OGL_Draw
- Signature: `void OGL_Draw(SDL_Rect& dst)`
- Purpose: Render shape via OpenGL using TextureManager, handling four texture types with different coordinate/cropping logic.
- Inputs: Destination SDL_Rect (screen position and size)
- Outputs/Return: None (renders to OpenGL context)
- Side effects: Sets OpenGL state (glEnable, glBlendFunc, texture matrix, modelview matrix); reads/modifies TextureManager; may render glow maps
- Calls: `get_shape_bitmap_and_shading_table`, `View_GetLandscapeOptions`, `TMgr.Setup()`, `TMgr.SetupTextureMatrix()`, `TMgr.RenderNormal()`, `TMgr.IsGlowMapped()`, `TMgr.RenderGlowing()`, `TMgr.RestoreTextureMatrix()`, OpenGL calls (`glEnable`, `glBlendFunc`, `glMatrixMode`, `glPushMatrix`, `glTranslatef`, `glRotatef`, `glBegin`, `glTexCoord2d`, `glVertex2i`, `glEnd`, `glPopMatrix`)
- Notes: Conditional on `HAVE_OPENGL`. Handles rotation via 2D matrix transform. Landscape and Interface types have custom U/V scaling and cropping logic; other types share a common path. Rotation is applied if magnitude > 0.1 degrees.

### SDL_Draw
- Signature: `void SDL_Draw(SDL_Surface *dst_surface, SDL_Rect& dst)`
- Purpose: Render shape via SDL software blitter, applying transformations and caching scaled/rotated surfaces.
- Inputs: Destination SDL_Surface pointer, destination SDL_Rect (screen bounds)
- Outputs/Return: None (blits to dst_surface)
- Side effects: Lazy-loads m_surface from shape resource; caches/reallocates m_scaled_surface; calls `rescale_surface`, `rotate_surface`, `flip_surface`; invokes `SDL_BlitSurface`
- Calls: `get_shape_surface`, `SDL_DisplayFormatAlpha`, `SDL_FreeSurface`, `free`, `rescale_surface`, `rotate_surface`, `flip_surface`, `SDL_BlitSurface`
- Notes: Early return if dst_surface is NULL. Surface is loaded once and cached. Scaled surface is regenerated if size mismatch detected. Walls are rotated; landscapes are flipped vertically. Crop rect is applied during blit.

### flip_surface
- Signature: `SDL_Surface *flip_surface(SDL_Surface *s, int width, int height)`
- Purpose: Create a vertically-flipped copy of an SDL surface (used for landscape textures).
- Inputs: Source SDL_Surface, width and height for output surface
- Outputs/Return: Newly-allocated SDL_Surface (caller must free)
- Side effects: Allocates new SDL_Surface and copies palette if present
- Calls: `SDL_CreateRGBSurface`, `SDL_SetColors`
- Notes: Early return NULL if input is NULL. Reverses scanline order (y -> height - y - 1). Preserves pixel format and palette.

### Destructor
- Signature: `~Shape_Blitter()`
- Purpose: Release SDL surface resources.
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `SDL_FreeSurface` on m_surface and m_scaled_surface (if different from m_surface)
- Calls: `SDL_FreeSurface`
- Notes: Safe to call if surfaces are NULL. Avoids double-free by checking m_scaled_surface != m_surface before freeing scaled version.

## Control Flow Notes
**Initialization:** Constructor loads shape descriptor and dimensions via `get_shape_surface`, establishing m_src bounds. Surfaces remain NULL.

**Render Phase:** Two independent code paths exist:
- **OpenGL path** (OGL_Draw): Sets up TextureManager immediately, applies crop/rotation via texture coordinates and matrix transforms, renders GL_TRIANGLE_FAN primitives. No surface caching.
- **SDL path** (SDL_Draw): Lazily loads m_surface on first call, caches rescaled/transformed version in m_scaled_surface, blits via SDL_BlitSurface. Transformations persist across frames.

**Scaling:** Rescale adjusts m_scaled_src and proportionally recomputes crop_rect; does not trigger immediate surface regeneration (deferred to next SDL_Draw or ignored by OGL_Draw).

## External Dependencies
- **Notable includes:**
  - `Shape_Blitter.h` ΓÇô class definition
  - `interface.h` ΓÇô `get_shape_bitmap_and_shading_table`, shape descriptor macros
  - `render.h` ΓÇô rendering structures and transfer modes
  - `images.h` ΓÇô `rescale_surface`, SDL surface utilities
  - `shell.h` ΓÇô `get_shape_surface`
  - `scottish_textures.h` ΓÇô transfer modes, `_shading_normal`, `_shadeless_transfer`, `_tinted_transfer`
  - `OGL_Setup.h` ΓÇô OpenGL configuration, `SglColor4f`, `TextureManager`
  - `OGL_Textures.h` ΓÇô texture manager internals
  - `OGL_Blitter.h` ΓÇô OpenGL blitting utilities
  - `<GL/gl.h>`, `<GL/glu.h>`, `<GL/glext.h>` (conditional on platform and `HAVE_OPENGL`)
  - `<SDL/SDL.h>` ΓÇô SDL surface, rect, and pixel types

- **Defined elsewhere:** `rotate_surface` (declared; implemented in `HUDRenderer_SW.cpp`), `rescale_surface`, `get_shape_surface`, `get_shape_bitmap_and_shading_table`, `View_GetLandscapeOptions`, OpenGL context and matrix state
