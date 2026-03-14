# Source_Files/RenderMain/RenderRasterize.h

## File Purpose
Defines the `RenderRasterizerClass` for converting sorted world geometry into screen-space rendering commands. Acts as the bridge between the visibility tree / polygon sorting phases and the backend rasterizer, handling clipping and coordinate transformations for horizontal surfaces (floors/ceilings), vertical surfaces (walls), and objects/sprites.

## Core Responsibilities
- Rasterize prepared geometry from the visibility tree into draw calls
- Clip horizontal polygons (floors/ceilings) in XY and Z dimensions
- Clip vertical polygons (walls) in XZ dimension
- Render floors, ceilings, walls with lighting and texture information
- Render sprites/objects with proper depth ordering and media boundary handling
- Maintain coordinate transformations between world space and clipping-aware screen space

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `flagged_world_point2d` | struct | 2D world point with clipping flags; used for floor/ceiling polygon vertices; flags track which screen edges the point is clipped by |
| `flagged_world_point3d` | struct | 3D world point with clipping flags; used for ceiling vertices to handle height variation |
| `vertical_surface_data` | struct | Describes a wall surface with texture, lighting, height bounds (h0, h1, hmax), and endpoints (p0, p1) for screen transformation |
| `RenderRasterizerClass` | class | Main orchestrator; owns view data and pointers to polygon sorter and rasterizer backend; drives the frame's rasterization pass |

## Global / File-Static State
None.

## Key Functions / Methods

### render_node_floor_or_ceiling
- Signature: `void render_node_floor_or_ceiling(clipping_window_data *window, polygon_data *polygon, horizontal_surface_data *surface, bool void_present)`
- Purpose: Rasterize a floor or ceiling polygon
- Inputs: clipping window, polygon definition, surface data (texture/lighting), flag for void presence on opposite side
- Side effects: Emits draw calls via `RasPtr` backend; may read/write clipping state
- Notes: `void_present` flag suppresses semitransparency artifacts when rendering across void boundaries

### render_node_side
- Signature: `void render_node_side(clipping_window_data *window, vertical_surface_data *surface, bool void_present)`
- Purpose: Rasterize a wall surface (vertical side)
- Inputs: clipping window, wall surface definition, void presence flag
- Side effects: Emits draw calls via backend
- Notes: Handles transfer modes and height clipping

### render_node_object
- Signature: `void render_node_object(render_object_data *object, bool other_side_of_media)`
- Purpose: Rasterize a sprite/object (inhabitant)
- Inputs: object definition, flag indicating if on opposite side of liquid surface
- Side effects: Emits sprite draw call
- Notes: `other_side_of_media` distinguishes rendering above vs. below liquid boundaries

### Clipping Functions
Summary of 6 clipping helper methods (not detailed individually to preserve token budget):
- `xy_clip_horizontal_polygon()` / `xy_clip_flagged_world_points()`: Clip horizontal polygons to screen bounds (left/right/top/bottom)
- `z_clip_horizontal_polygon()` / `z_clip_flagged_world_points()`: Clip floors/ceilings by height threshold
- `xz_clip_vertical_polygon()` / `xz_clip_flagged_world_points()`: Clip walls in XZ plane
- `xy_clip_line()`: Clip a line segment in XY space

### render_tree
- Signature: `void render_tree()`
- Purpose: Execute the complete rendering pass for one frame
- Inputs: Uses member pointers (`view`, `RSPtr`, `RasPtr`)
- Side effects: Iterates sorted polygons and objects, emits all rendering commands for the frame
- Notes: Main entry point from frame loop; relies on prior polygon sorting via `RSPtr`

### RenderRasterizerClass
- Signature: `RenderRasterizerClass()`
- Purpose: Constructor; initializes the rasterizer orchestrator
- Side effects: Presumably initializes member pointers to null or valid references

## Control Flow Notes
This class sits in the **frameΓåÆsortΓåÆrasterize** pipeline:
1. **RenderVisTreeClass** builds visibility from camera position
2. **RenderSortPolyClass** (`RSPtr`) sorts polygons by depth
3. **RenderRasterizerClass** (this file) iterates sorted nodes and emits rasterization calls
4. **RasterizerClass** (`RasPtr`) backend (software or OpenGL) executes draw commands

`render_tree()` is called once per frame after `RSPtr->sort_render_tree()`.

## External Dependencies
- `<vector>` ΓÇô STL dynamic arrays
- `world.h` ΓÇô World coordinate types (`world_distance`, `long_vector2d`, point/vector structs); trigonometry; distance functions
- `render.h` ΓÇô View data (`view_data`), render flags, screen polygon/rectangle definitions
- `RenderSortPoly.h` ΓÇô `RenderSortPolyClass` for depth-sorted polygon lists
- `RenderPlaceObjs.h` ΓÇô `render_object_data` struct for sprite rendering
- `Rasterizer.h` ΓÇô Base class `RasterizerClass` (abstract backend); `SetView()`, texture/rectangle methods
