# Source_Files/RenderMain/NewRenderRasterize.h

## File Purpose

Header defining the `NewRenderRasterizer` class, which coordinates rendering of visibility-culled surfaces and objects into a rasterization backend. Refactors procedural rasterization code from `render.c` into a class-based architecture with centralized view and clipping state management.

## Core Responsibilities

- Coordinate per-frame rendering of horizontal surfaces (floors/ceilings), vertical surfaces (walls), and objects/sprites
- Manage clipping windows and viewport-space transformations
- Perform geometric clipping in 2D (XY), height (Z), and 3D (XZ) coordinate systems
- Delegate actual rasterization to a pluggable `RasterizerClass` backend
- Support long-distance world coordinates via `flagged_world_point2d/3d` structures
- Generate automap/minimap rendering via `fake_render_tree()`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `flagged_world_point2d` | struct | 2D world point with clipping flag bits for frustum culling state |
| `flagged_world_point3d` | struct | 3D world point (extends `flagged_world_point2d`) with z-height for vertical geometry |
| `vertical_surface_data` | struct | Wall/side definition: texture, height bounds, endpoints, ambient lighting delta |
| `NewRenderRasterizer` | class | Main coordinator class; holds view state, manages rendering pipeline |

## Global / File-Static State

None declared in this header.

## Key Functions / Methods

### render_tree
- **Signature:** `void render_tree(NewVisTree *tree, RenderPlaceObjsClass *objs)`
- **Purpose:** Main per-frame rendering entry point; processes visibility tree to rasterize all visible surfaces and objects
- **Inputs:** Visibility tree (computed frustum culling + portal visibility), placed object list
- **Outputs/Return:** None (side effects: delegates to `RasterizerClass`)
- **Side effects:** Calls underlying rasterizer methods via `rast`; reads/writes global render state
- **Notes:** Called once per frame during rendering pipeline

### fake_render_tree
- **Signature:** `void fake_render_tree(NewVisTree *tree)`
- **Purpose:** Generate automap/overhead-map rendering without full rasterization
- **Inputs:** Visibility tree
- **Outputs/Return:** None
- **Notes:** Parallel rendering path for UI overlays

### SetRasterizer
- **Signature:** `void SetRasterizer(RasterizerClass *Ras)`
- **Purpose:** Inject the concrete rasterizer implementation (software, OpenGL, etc.)
- **Inputs:** Rasterizer instance pointer
- **Notes:** Must be called before rendering; enables backend polymorphism

### SetView
- **Signature:** `void SetView(view_data *View)`
- **Purpose:** Set the current camera view state (position, orientation, FOV)
- **Inputs:** View structure with camera parameters
- **Notes:** Must be called before `render_tree()`

### Clipping methods (private)
Five clipping functions handle polygon/point clipping in different coordinate spaces:
- `xy_clip_horizontal_polygon()` / `xy_clip_flagged_world_points()` ΓÇö 2D screen-space clipping
- `z_clip_horizontal_polygon()` / `z_clip_flagged_world_points()` ΓÇö height/depth clipping for ceilings
- `xz_clip_vertical_polygon()` / `xz_clip_flagged_world_points()` ΓÇö 3D depth clipping for walls
- `xy_clip_line()` ΓÇö edge line clipping

All take clipping window, geometry, and clipping line parameters; return clipped vertex count or modified point.

### render_node_* (private)
- `render_node_floor_or_ceiling()` ΓÇö rasterize horizontal polygons with optional void-side transparency handling
- `render_node_side()` ΓÇö rasterize vertical polygons (walls)
- `render_node_object()` ΓÇö rasterize sprites/objects with vertical clip bounds and portal view context
- `render_clip_debug_lines()` ΓÇö debug visualization

## Control Flow Notes

Fits into per-frame rendering pipeline after visibility determination:
1. **Visibility phase** ΓåÆ produces `NewVisTree` with front-to-back sorted nodes and portal views
2. **Object placement phase** ΓåÆ produces `RenderPlaceObjsClass` with sprite positions
3. **Rasterization phase** (this class) ΓåÆ iterates visibility tree, clips to frustum/portals, delegates to backend rasterizer
4. **Output** ΓåÆ framebuffer via `RasterizerClass` (software blit, OpenGL draw calls, etc.)

The `fake_render_tree()` path branches for automap generation without full rasterization.

## External Dependencies

- **`world.h`** ΓÇö world geometry types: `world_distance`, `long_point2d`, `long_vector2d`, angle constants
- **`render.h`** ΓÇö `view_data` (camera state), render flags, viewport constants
- **`NewRenderVisTree.h`** ΓÇö `NewVisTree`, `clipping_window_data`, `portal_view_data`, `render_node_data`, `translated_endpoint_data`
- **`RenderPlaceObjs.h`** ΓÇö `RenderPlaceObjsClass`, `render_object_data`
- **`Rasterizer.h`** ΓÇö `RasterizerClass` (abstract backend)
- **`<vector>`** ΓÇö STL dynamic arrays (indirectly used via member pointers to render data)

**Symbols defined elsewhere:**
- `polygon_data`, `horizontal_surface_data`, `render_object_data` ΓÇö defined in included headers or elsewhere in render subsystem
- `bitmap_definition` ΓÇö frame/texture target (used in bundled `render.h`)
