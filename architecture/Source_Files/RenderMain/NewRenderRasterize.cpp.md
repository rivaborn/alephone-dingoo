# Source_Files/RenderMain/NewRenderRasterize.cpp

## File Purpose
Implements polygon rasterization and clipping for the game renderer. Transforms 3D world geometry into 2D screen coordinates, clips polygons to viewport boundaries, and manages the rendering of walls, floors, ceilings, and liquid surfaces from a preprocessed visibility tree.

## Core Responsibilities
- Traverse and render the visibility tree of polygons sorted front-to-back
- Transform world-space vertex coordinates to screen-space via perspective projection
- Clip polygons to clipping windows (viewport boundaries and portal boundaries)
- Render vertical surfaces (walls/sides) with ambient lighting and texture mapping
- Render horizontal surfaces (floors/ceilings) with texture mapping
- Render liquid surface layers with support for transparency
- Generate automap data through a simplified render pass
- Support animated textures and transfer modes (fade, invisibility, static, etc.)
- Manage rasterization windows and apply debug visualization

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `flagged_world_point2d` | struct | 2D world coordinate with clipping flags for boundary testing |
| `flagged_world_point3d` | struct | 3D world coordinate extending 2D with height; used for wall clipping |
| `vertical_surface_data` | struct | Wall/side surface: position, height range, texture definition, ambient adjustment |
| `NewRenderRasterizer` | class | Main rasterizer class; holds view data and rendering methods |
| `polygon_definition` | struct | Textured polygon ready for rasterization; vertex list, texture, shading |
| `rasterize_window` | struct (external) | Rectangular region for rasterization on screen |
| `clipping_window_data` | struct (external) | Viewport boundary specification; cascaded clipping windows |
| `horizontal_surface_data` | struct (external) | Floor/ceiling surface: height, texture, lightsource, transfer mode |
| `portal_view_data` | struct (external) | View origin and parameters for a portal/area |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rasterize_windows` | `static rasterize_window[100]` | static | Scratch array holding clipping windows for current node; reused per polygon |

## Key Functions / Methods

### render_tree
- **Signature:** `void render_tree(NewVisTree *tree, RenderPlaceObjsClass *objs)`
- **Purpose:** Main rendering pipeline; processes visibility tree and renders all visible polygons in depth order
- **Inputs:** Visibility tree (sorted nodes, endpoints, clipping windows), object placement data
- **Outputs/Return:** None (side effect: rasterized to frame buffer via rasterizer)
- **Side effects:** Updates global state (nodal references); invokes clipping and texturing
- **Calls:** `checkPortalViewIndex()`, `render_node_side()`, `render_node_floor_or_ceiling()`, `render_clip_debug_lines()`, AnimTxtr_Translate()
- **Notes:** Loops over sorted nodes back-to-front; handles all three side types (_full_side, _split_side, _high_side, _low_side); renders transparent textures separately; liquid surfaces rendered between background and foreground objects; gates liquid surface rendering on media height bounds.

### fake_render_tree
- **Signature:** `void fake_render_tree(NewVisTree *tree)`
- **Purpose:** Non-rendering tree traversal to populate automap; marks walls and polygons as explored
- **Inputs:** Visibility tree
- **Outputs/Return:** None (updates global automap state)
- **Side effects:** Calls `ADD_POLYGON_TO_AUTOMAP()` and `ADD_LINE_TO_AUTOMAP()` macros
- **Calls:** `checkPortalViewIndex()`
- **Notes:** Skipped if polygons are not tall enough (floor above ceiling); used to build map annotations without rendering overhead

### render_node_floor_or_ceiling
- **Signature:** `void render_node_floor_or_ceiling(clipping_window_data *vis_extents, int rasterize_windows_used, polygon_data *polygon, horizontal_surface_data *surface, bool void_present, portal_view_data *portalView)`
- **Purpose:** Rasterize a floor or ceiling polygon with perspective-correct texture mapping
- **Inputs:** Clipping window extents, window count, polygon and surface data, void presence flag, view data
- **Outputs/Return:** None (rasterizes via `rast->texture_horizontal_polygon()`)
- **Side effects:** Transforms and clips vertex list; calls AnimTxtr_Translate(), get_shape_bitmap_and_shading_table(), instantiate_polygon_transfer_mode()
- **Calls:** `xy_clip_horizontal_polygon()`, `z_clip_horizontal_polygon()`, `get_endpoint_data()`, AnimTxtr_Translate(), get_light_intensity()
- **Notes:** Returns early if no vertices remain after clipping; reverses vertex order for ceiling polygons; clips to left/right then top/bottom (order matters); applies transfer modes and infravision shading

### render_node_side
- **Signature:** `void render_node_side(clipping_window_data *vis_extents, int rasterize_windows_used, vertical_surface_data *surface, bool void_present, portal_view_data *portalView)`
- **Purpose:** Rasterize a wall or side polygon with perspective-correct texture mapping
- **Inputs:** Clipping extents, window count, vertical surface data, void flag, view data
- **Outputs/Return:** None (rasterizes via `rast->texture_vertical_polygon()`)
- **Side effects:** Transforms 3D endpoints to screen; clips vertices; applies ambient shading and transfer mode
- **Calls:** `xy_clip_line()`, `xz_clip_vertical_polygon()`, AnimTxtr_Translate(), get_shape_bitmap_and_shading_table(), instantiate_polygon_transfer_mode()
- **Notes:** Skips if surface faces away from camera (clockwise test); constructs trapezoid from two posts (endpoints); clips in xz plane (top/bottom of wall); front-face culling via cross-product check; handles ambient delta clamping to [0, FIXED_ONE]

### xy_clip_horizontal_polygon
- **Signature:** `short xy_clip_horizontal_polygon(flagged_world_point2d *vertices, short vertex_count, long_vector2d *line, uint16 flag)`
- **Purpose:** Sutherland-Hodgman clipping of 2D polygon to a half-plane defined by a line
- **Inputs:** Vertex array, count, clipping line (normal vector), clip flag to mark clipped vertices
- **Outputs/Return:** New vertex count after clipping
- **Side effects:** Modifies vertex array in-place; inserts new vertices at intersection points; uses memmove() for array compaction
- **Calls:** `xy_clip_flagged_world_points()`, memmove()
- **Notes:** State machine tracks entrance/exit points; handles polygons wholly inside/outside; clips vertices only if marked clipped_*_vertex flags

### z_clip_horizontal_polygon
- **Signature:** `short z_clip_horizontal_polygon(flagged_world_point2d *vertices, short vertex_count, long_vector2d *line, world_distance height, uint16 flag)`
- **Purpose:** Clips 2D polygon to a height constraint (used for floor/ceiling bounds)
- **Inputs:** Vertex array, count, height clipping line, height value, clip flag
- **Outputs/Return:** New vertex count
- **Side effects:** Modifies vertex array; inserts clipped vertices
- **Calls:** `z_clip_flagged_world_points()`, memmove()
- **Notes:** Variant of xy_clip; line encodes height constraint; cross_product computed as `heighti - line->j * x` where `heighti = line->i * height`

### xz_clip_vertical_polygon
- **Signature:** `short xz_clip_vertical_polygon(flagged_world_point3d *vertices, short vertex_count, long_vector2d *line, uint16 flag)`
- **Purpose:** Sutherland-Hodgman clipping of 3D polygon to a vertical plane (xz-plane)
- **Inputs:** 3D vertex array, count, clipping plane normal (as 2D line in xz), clip flag
- **Outputs/Return:** New vertex count
- **Side effects:** Modifies 3D vertex array; inserts new 3D clipped vertices
- **Calls:** `xz_clip_flagged_world_points()`, memmove()
- **Notes:** Cross-product in xz plane: `line->i * z - line->j * x`; state machine identical to xy variant; used for top/bottom viewport clipping of walls

### Clipping helper functions
- **xy_clip_flagged_world_points, z_clip_flagged_world_points, xz_clip_flagged_world_points**: Compute intersection of edge with clipping plane using fixed-point arithmetic with bit-shifting to maintain precision
- **xy_clip_line**: Two-point clipping for line segments (used for wall endpoints)

## Control Flow Notes
- **Frame entry:** `render_tree()` is called once per frame with the visibility tree
- **Per-polygon loop:** For each sorted node, extract polygon and portal view; build clipping window extents by cascading window hierarchy
- **Rendering sequence:** Render sides/walls ΓåÆ render ceiling (if above viewer) ΓåÆ render floor (if below viewer) ΓåÆ render liquid surface
- **Per-surface rendering:** Transform vertices ΓåÆ clip to cascaded windows ΓåÆ transform to screen-space ΓåÆ invoke rasterizer
- **Automap:** `fake_render_tree()` called separately to mark explored areas without rendering

## External Dependencies
- **Includes (game world):** `map.h` (polygons, sides, endpoints, lines), `lightsource.h` (light intensity), `media.h` (liquid surfaces)
- **Includes (rendering):** `NewRenderRasterize.h` (class definition), `AnimatedTextures.h` (texture animation), `OGL_Setup.h` (OpenGL config, shading tables), `Rasterizer.h` (rasterizer interface)
- **Includes (base):** `cseries.h` (platform macros, types)
- **Symbols defined elsewhere:** `get_line_data()`, `get_side_data()`, `get_endpoint_data()`, `get_media_data()` (accessors), `get_shape_bitmap_and_shading_table()`, `instantiate_polygon_transfer_mode()` (texture setup), `AnimTxtr_Translate()` (animated texture), `get_light_intensity()` (lighting), `rast->texture_horizontal_polygon()`, `rast->texture_vertical_polygon()`, `rast->debug_line_v()`, `rast->debug_line_h()` (rasterizer backend), `map_polygons` (global polygon array)
