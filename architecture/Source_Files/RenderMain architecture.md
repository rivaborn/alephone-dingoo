# Subsystem Overview

## Purpose

RenderMain coordinates the Marathon game engine's 3D-to-2D rendering pipeline, transforming world geometry (polygons, walls, floors, objects, sprites) into a sorted, rasterized 2D image. It constructs visibility trees to cull occluded surfaces, sorts visible geometry by depth, places sprites and objects in correct rendering order, and rasterizes textured surfaces with lighting, transfer modes, and animated textures to the framebuffer.

## Key Files

| File | Role |
|------|------|
| render.cpp | Central render frame coordinator; orchestrates visibility, sorting, object placement, rasterization via specialized classes |
| render.h | Core interface; defines view_data (camera parameters), rendering constants, main render_view() entry point |
| NewRenderVisTree.cpp / NewRenderVisTree.h | Builds hierarchical visibility tree via portal-based ray-casting; determines which polygons are visible from viewpoint |
| RenderSortPoly.cpp / RenderSortPoly.h | Decomposes visibility tree into depth-sorted polygon list with screen-space clipping windows |
| RenderPlaceObjs.cpp / RenderPlaceObjs.h | Places game objects (sprites, monsters, items) in sorted depth order; computes screen projections and bounding boxes |
| RenderRasterize.cpp / RenderRasterize.h | Clips polygons to viewport and clipping windows; dispatches textured surfaces to rasterizer backend |
| Rasterizer.h | Abstract rasterizer interface; backend implementations override SetView(), texture_horizontal_polygon(), texture_vertical_polygon() |
| Rasterizer_SW.h | Software rasterizer; delegates to scottish_textures module |
| Rasterizer_OGL.h | OpenGL rasterizer wrapper (conditional on HAVE_OPENGL) |
| scottish_textures.cpp / scottish_textures.h | Software polygon texture mapping with DDA coordinate interpolation; supports 8/16/32-bit pixel formats and transfer modes |
| texturers.cpp / texturers.h | Template instantiation hub; generates specialized texture functions for different color depths and transparency modes |
| textures.cpp / textures.h | Bitmap metadata and row-address pre-computation for both linear and RLE-compressed bitmap formats |
| AnimatedTextures.cpp / AnimatedTextures.h | Maintains per-collection animated texture state; translates static texture descriptors to current animation frame |
| shapes.cpp | Loads graphics collections from disk; builds shading tables and manages color lookup tables (CLUTs) |
| shape_descriptors.h | Defines packed 16-bit shape_descriptor bitfield (collection ID, shape index, CLUT); provides extraction/construction macros |
| collection_definition.h | Defines binary data structures for sprite/texture collections (palettes, animation definitions, bitmap layout) |
| OGL_Setup.cpp / OGL_Setup.h | OpenGL initialization, detection, configuration; manages texture quality tiers and model loading (disabled on Dingoo) |
| OGL_Model_Def.cpp / OGL_Model_Def.h | 3D model definitions, skins, and GPU texture management for OpenGL backend |
| OGL_Render.cpp / OGL_Render.h | Main OpenGL rendering interface; projection/view matrices, 3D model and UI rendering (disabled on Dingoo) |
| OGL_Textures.cpp / OGL_Textures.h | OpenGL texture manager; loading, caching, format conversion, infravision/silhouette effects (disabled on Dingoo) |
| Crosshairs.cpp / Crosshairs_SDL.cpp / Crosshairs.h | HUD crosshair rendering; supports multiple shapes (cross, circle) with configurable size and color |
| ImageLoader.h / ImageLoader_SDL.cpp / ImageLoader_Shared.cpp | Image file loading (DDS, PNG, JPEG via SDL_image); mipmap management and DXTC decompression |
| low_level_textures.h | Template-based pixel blending and scanline texture mapping for horizontal/vertical polygons across bit depths |
| SW_Texture_Extras.cpp / SW_Texture_Extras.h | Software texture opacity tables for per-pixel alpha blending; disabled on Dingoo (HAVE_DINGOO) for binary size |
| WorkQueue.h | Template FIFO queue with object pooling; used by visibility tree generator |
| DDS.h | DirectDraw Surface file format specification (structure and constant definitions) |

## Core Responsibilities

- **Visibility determination**: Build portal-based visibility tree via ray-casting; determine which polygons are visible from player viewpoint
- **Depth sorting**: Decompose visibility tree into back-to-front sorted polygon list with screen-space clipping regions
- **Object placement**: Place sprites and game objects in sorted depth order; compute 2D screen projections and 3D model bounding boxes
- **Polygon rasterization**: Clip textured polygons to viewport and clipping windows; dispatch to hardware rasterizer backend
- **Texture mapping**: Interpolate texture coordinates via DDA; support perspective-correct mapping, animated textures, and transfer modes
- **Lighting and effects**: Apply shading tables for distance/angle lighting; instantiate transfer modes (fade, invisibility, static, pulsate, wobble)
- **Asset management**: Load and cache sprite/texture collections; manage color lookup tables, shading tables, and animated texture state
- **Multi-format support**: Render 8/16/32-bit pixel depths; decompress RLE and DXTC compressed bitmap formats
- **HUD rendering**: Render crosshairs and UI overlays in screen-space coordinates

## Key Interfaces & Data Flow

**Consumes from other subsystems:**
- World geometry (map.h): `polygon_data`, `side_data`, `endpoint_data`, `line_data` for visible surface construction
- Objects and sprites (map.h): Game object positions, types, animation states for depth-sorted placement
- Light sources (lightsource.h): Intensity lookups for per-surface shading calculations
- Media/liquids (media.h): Boundary checks for liquid surface transparency and viewer-relative clipping
- View camera (render.h, ViewControl.h): FOV, position, orientation, viewport dimensions, effects state
- Animated textures (AnimatedTextures.h): Per-frame texture animation updates and frame translation
- Preferences (preferences.h): Rendering mode (software vs. OpenGL), alpha blending mode, graphics quality
- Weapons display (weapons.h): Weapon sprite and animation data for foreground rendering

**Exposes to other subsystems:**
- Framebuffer (SDL surface or OpenGL context): Final 2D rasterized image to be displayed
- HUD/crosshairs: Screen-space overlay coordinates and rendering targets
- Progress callbacks: Asset loading progress during collection initialization (OpenGL backend only)

**Architecture pattern:**
- Dispatcher (render.cpp) ΓåÆ Visibility (RenderVisTree) ΓåÆ Sorting (RenderSortPoly) ΓåÆ Object Placement (RenderPlaceObjs) ΓåÆ Rasterization (RenderRasterize) ΓåÆ Backend (Rasterizer_SW or Rasterizer_OGL)
- All coordinate transformations occur between world-space (map geometry) and screen-space (viewport)

## Runtime Role

**Initialization:**
- Load sprite/texture collections from disk; build shading tables and color lookup tables
- OpenGL backend (if enabled): Initialize context, detect extensions, pre-load textures
- Allocate rasterizer state and temporary polygon/clipping data buffers

**Per-frame rendering:**
1. Clear framebuffer and reset render flags on map geometry
2. Construct visibility tree from player's current location and viewpoint
3. Sort visible polygons by depth (back-to-front)
4. Place game objects (sprites, monsters) in depth order with 2D screen projections
5. Rasterize sorted geometry via polygon clipping and texture mapping to selected backend
6. Render foreground weapon/HUD layer in screen-space
7. Render HUD overlays (crosshairs, text, infravision tint)
8. Update animated texture state for next frame

**Shutdown:**
- Unload sprite/texture collections; deallocate shading tables
- OpenGL backend: Delete textures, unload 3D models, destroy rendering context
- Free rasterizer state and temporary buffers

## Notable Implementation Details

- **Dual visibility tree implementations:** RenderVisTree.cpp (legacy) and NewRenderVisTree.cpp (refactored) indicate architectural evolution; new version uses class-based state management
- **Backend abstraction:** Rasterizer.h base class allows software and OpenGL implementations to coexist; render.cpp selects at initialization via conditional compilation
- **Software rasterizer template instantiation:** texturers.cpp includes low_level_textures.cpp multiple times with different preprocessor configurations to generate specialized functions for 8/16/32-bit color depths and transparency modes; avoids runtime dispatching overhead
- **Coordinate overflow handling:** NewRenderVisTree uses `long_point2d` (extended-precision 2D vectors) to prevent overflow on long-distance ray-casting calculations in large game worlds
- **Dingoo-specific optimizations:** SW_Texture_Extras.h conditionally disabled (HAVE_DINGOO guard) to reduce binary size; OpenGL subsystem entirely excluded on Dingoo (no HAVE_OPENGL), forcing software-only rendering on 400 MHz / 32 MB Ingenic JZ4740 hardware; 320├ù240 fixed resolution baked into render constants
- **RLE and DXTC decompression:** ImageLoader_Shared.cpp handles on-the-fly decompression of RLE and DXTC1/3/5 compressed bitmap formats; avoids large uncompressed texture memory footprint
- **Polygon clipping state machine:** RenderRasterize.cpp uses procedural clipping in XY, Z, and XZ coordinate spaces to handle viewport frustum and portal boundary clipping
- **Landscape texture optimization:** scottish_textures.cpp includes separate landscape rendering path with fixed-scale texture coordinate sampling and configurable aspect ratios to efficiently render large repeating terrain surfaces
