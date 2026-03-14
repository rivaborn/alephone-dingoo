# Source_Files/RenderMain/RenderRasterize.cpp

## File Purpose
Implements polygon clipping and rasterization for the Marathon-compatible game engine's rendering pipeline. Transforms world-space geometry into screen-space polygons, handles visibility culling via clipping windows, and dispatches final rendering to a hardware rasterizer (software or OpenGL).

## Core Responsibilities
- Iterate sorted BSP nodes and render visible geometry (walls, floors, ceilings)
- Manage horizontal (floor/ceiling) and vertical (wall) surface rendering with correct height culling
- Handle semitransparent liquid surfaces and viewer position relative to media boundaries
- Clip polygons against viewing frustum and window boundaries (xy, xz, z coordinate spaces)
- Calculate texture origins, lighting deltas, and transfer modes for each surface
- Dispatch textured polygons to a hardware rasterizer object
- Manage vertex lists and clipping state through state-machine-based polygon clipping

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `flagged_world_point2d` | struct | 2D world point with clipping-status flags (for horizontal surfaces) |
| `flagged_world_point3d` | struct | 3D world point with clipping-status flags (for vertical surfaces) |
| `vertical_surface_data` | struct | Parameterizes a wall side: endpoints, heights, texture, lighting |
| `RenderRasterizerClass` | class | Main rasterizer: holds view, sorted polygons, and rasterizer target |

## Global / File-Static State
None (all state is instance-based in `RenderRasterizerClass`).

## Key Functions / Methods

### render_tree
- **Signature:** `void RenderRasterizerClass::render_tree()`
- **Purpose:** Main entry point; iterates sorted BSP nodes and renders all visible geometry (walls, floors, ceilings, objects, liquids).
- **Inputs:** None; uses member `RSPtr->SortedNodes` (vector of sorted nodes), member `view` (viewer state).
- **Outputs/Return:** None; side effect is dispatch of textured polygons to `RasPtr` (rasterizer).
- **Side effects:** Updates member state (view), calls render helpers. Reads world data (`get_polygon_data`, `get_media_data`, etc.). Dispatches `RasPtr->texture_horizontal_polygon()` and `RasPtr->texture_vertical_polygon()`.
- **Calls (direct):**
  - `render_node_floor_or_ceiling()`, `render_node_side()`, `render_node_object()`
  - World accessors: `get_polygon_data()`, `get_media_data()`, `get_line_data()`, `get_side_data()`, `get_endpoint_data()`
  - `AnimTxtr_Translate()`, `get_light_intensity()`, `get_shape_bitmap_and_shading_table()`, `instantiate_polygon_transfer_mode()`
- **Notes:**
  - State machine for media liquid visibility (semitransparent vs. opaque).
  - Suppresses void texture for full-side walls with adjacent polygons; special handling for split-sides (high/low).
  - Conditional rendering of liquid surfaces only if semitransparent and between floor/ceiling.

### render_node_floor_or_ceiling
- **Signature:** `void RenderRasterizerClass::render_node_floor_or_ceiling(clipping_window_data *window, polygon_data *polygon, horizontal_surface_data *surface, bool void_present)`
- **Purpose:** Render a floor or ceiling polygon with texture, clipping, and perspective-correct transformation.
- **Inputs:** Clipping window, polygon geometry, horizontal surface (texture, height, lighting), flag indicating void on far side.
- **Outputs/Return:** None; dispatches textured polygon via `RasPtr->texture_horizontal_polygon()`.
- **Side effects:** Allocates local vertex buffer, clips to window bounds, transforms to screen space, sets up texture and shading.
- **Calls:**
  - `xy_clip_horizontal_polygon()` (left/right clipping), `z_clip_horizontal_polygon()` (top/bottom clipping)
  - `AnimTxtr_Translate()`, `get_shape_bitmap_and_shading_table()`, `get_light_intensity()`, `instantiate_polygon_transfer_mode()`
- **Notes:**
  - Reverses vertex order for ceiling (adjusted_height > 0).
  - Long-distance friendly: guards against division by zero in screen-space projection.
  - Sets `VoidPresent` flag to suppress semitransparency artifacts.

### render_node_side
- **Signature:** `void RenderRasterizerClass::render_node_side(clipping_window_data *window, vertical_surface_data *surface, bool void_present)`
- **Purpose:** Render a wall side (vertical surface) with texture mapping and height clipping.
- **Inputs:** Clipping window, vertical surface (endpoints, heights, texture, lighting), void-presence flag.
- **Outputs/Return:** None; dispatches textured polygon via `RasPtr->texture_vertical_polygon()`.
- **Side effects:** Builds trapezoid from two posts, clips to window bounds (xz plane), transforms to screen space, computes texture vector and origin.
- **Calls:**
  - `xy_clip_line()`, `xz_clip_vertical_polygon()` (height-based clipping)
  - `AnimTxtr_Translate()`, `get_shape_bitmap_and_shading_table()`, `instantiate_polygon_transfer_mode()`
- **Notes:**
  - Handles three side types: full, high, low, split (with secondary texture for lower/upper panels).
  - Transparent texture rendering (if present) with separate height range.
  - Texture coordinates and scaling computed from `texture_definition->x0/y0` and side length.

### render_node_object
- **Signature:** `void RenderRasterizerClass::render_node_object(render_object_data *object, bool other_side_of_media)`
- **Purpose:** Render a 3D object (sprite/model) with liquid-boundary clipping.
- **Inputs:** Object render data, flag indicating if object is on opposite side of liquid surface from viewer.
- **Outputs/Return:** None; dispatches rectangle via `RasPtr->texture_rectangle()`.
- **Side effects:** Updates object clipping rectangle based on viewer position and media boundary.
- **Calls:** `RasPtr->texture_rectangle()`
- **Notes:**
  - Clips object against liquid surface if present (XOR logic between viewer position and object side).
  - Sets `BelowLiquid` flag for 3D models to handle self-contained clipping.

### xy_clip_horizontal_polygon
- **Signature:** `short RenderRasterizerClass::xy_clip_horizontal_polygon(flagged_world_point2d *vertices, short vertex_count, long_vector2d *line, uint16 flag)`
- **Purpose:** Clip a 2D polygon to a line boundary in xy-plane; used for left/right window clipping.
- **Inputs:** Vertex buffer, count, line (defines clipping plane via cross product), flag to mark clipped vertices.
- **Outputs/Return:** New vertex count after clipping.
- **Side effects:** Modifies vertex buffer in place; inserts new vertices at clip points.
- **Calls:** `xy_clip_flagged_world_points()`, `memmove()`
- **Notes:**
  - State machine: test first vertex, search for in/out and out/in transitions, clip and reorder.
  - Handles edge case of polygon fully inside or outside.

### z_clip_horizontal_polygon
- **Signature:** `short RenderRasterizerClass::z_clip_horizontal_polygon(flagged_world_point2d *vertices, short vertex_count, long_vector2d *line, world_distance height, uint16 flag)`
- **Purpose:** Clip a 2D polygon at a fixed height z; used for top/bottom window clipping.
- **Inputs:** Vertex buffer, count, line parameter (x component of clip), height, flag.
- **Outputs/Return:** New vertex count.
- **Side effects:** Modifies vertex buffer; inserts clipped vertices.
- **Calls:** `z_clip_flagged_world_points()`, `memmove()`

### xz_clip_vertical_polygon
- **Signature:** `short RenderRasterizerClass::xz_clip_vertical_polygon(flagged_world_point3d *vertices, short vertex_count, long_vector2d *line, uint16 flag)`
- **Purpose:** Clip a 3D polygon in xz-plane; used for top/bottom clipping of wall sides.
- **Inputs:** 3D vertex buffer, count, line (xz boundary), flag.
- **Outputs/Return:** New vertex count.
- **Side effects:** Modifies vertex buffer.
- **Calls:** `xz_clip_flagged_world_points()`, `memmove()`

### Clip helper functions (xy_clip_flagged_world_points, z_clip_flagged_world_points, xz_clip_flagged_world_points)
- **Purpose:** Compute the intersection of a line segment with a clipping boundary.
- **Notes:** Use fixed-point arithmetic with 16-bit shifts to prevent overflow on long-distance geometry. Preserve clipping flags via bitwise AND.

### xy_clip_line
- **Purpose:** Clip a 2D line segment (two endpoints); simpler version for post pairs in vertical surfaces.

## Control Flow Notes
- **Frame timing:** `render_tree()` is called once per frame during the render phase.
- **Polygon iteration:** Walks `RSPtr->SortedNodes` (back-to-front order for correct overdraw).
- **Surface setup:** Extracts and caches floor/ceiling/media surfaces to avoid repeated data lookups.
- **Window clipping:** For each clipping window, renders ceilings, walls (per vertex), and floors in order.
- **Liquid handling:** Conditional logic routes rendering based on viewer position relative to media boundary and semitransparency flag.
- **Post-render:** Final dispatch to `RasPtr` (hardware rasterizer) performs screen-space rasterization and pixel filling.

## External Dependencies
- **Includes:**
  - `map.h` ΓÇö polygon, line, side, endpoint, media data structures; world geometry accessors
  - `lightsource.h` ΓÇö light data and intensity queries
  - `media.h` ΓÇö media (liquid) data
  - `RenderRasterize.h` ΓÇö class definition and helper types
  - `AnimatedTextures.h` ΓÇö animated texture translation
  - `OGL_Setup.h` ΓÇö OpenGL configuration and color/bitmap helper functions
  - `preferences.h` ΓÇö graphics preferences
  - `screen.h` ΓÇö screen mode and rendering constants
- **Defined elsewhere:**
  - World accessor functions: `get_polygon_data()`, `get_media_data()`, `get_line_data()`, `get_side_data()`, `get_endpoint_data()` (from map.h)
  - `get_light_intensity()`, `get_light_data()` (from lightsource.h)
  - `AnimTxtr_Translate()` (animated texture manager)
  - `get_shape_bitmap_and_shading_table()`, `instantiate_polygon_transfer_mode()` (rendering setup)
  - `RasterizerClass` methods: `texture_horizontal_polygon()`, `texture_vertical_polygon()`, `texture_rectangle()` (dispatch to hardware)
  - Macro helpers: `TEST_RENDER_FLAG()`, `WRAP_HIGH()`, `WRAP_LOW()`, `MAX()`, `MIN()`, `PIN()`, `SGN()`, `overflow_short_to_long_2d()`
