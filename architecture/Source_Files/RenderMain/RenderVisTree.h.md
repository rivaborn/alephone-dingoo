# Source_Files/RenderMain/RenderVisTree.h

## File Purpose
Defines the `RenderVisTreeClass` and supporting data structures for calculating rendering visibility in the Aleph One game engine. This class builds a visibility tree to determine which map polygons (sectors/walls) are visible from a viewpoint, which gates the entire rendering pipeline. It manages polygon queues, clipping data for screen boundaries, and the BSP-like traversal of the game world's polygon adjacency graph.

## Core Responsibilities
- Build and maintain a visibility tree of polygons seen from the player viewpoint
- Cast visibility rays to determine which map polygons are in the view cone
- Track and calculate clipping information for screen boundaries (left, right, top, bottom edges)
- Translate map coordinates to screen-space coordinates for visible geometry
- Manage endpoint and line clipping data to prevent off-screen rendering
- Recursively traverse the map's polygon adjacency graph during ray casting
- Maintain a polygon queue for breadth-first visibility traversal

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `node_data` | struct | Tree node for a visible polygon; stores polygon index, clipping endpoint/line indices, parent/child/sibling pointers, and polygon-sort BSP links (PS_Greater, PS_Less, PS_Shared) |
| `line_clip_data` | struct | Clipping bounds and vectors for a screen-clipped map line; stores flags, x bounds, and top/bottom screen-space vectors |
| `endpoint_clip_data` | struct | Clipping information for a screen-clipped map endpoint; stores flags, screen x coordinate, and a long vector |
| `clipping_window_data` | struct | Rectangular clipping region; stores left/right/top/bottom vectors and linked-list next pointer |
| `RenderVisTreeClass` | class | Main visibility tree manager; owns node tree, polygon queue, clipping data, endpoint coordinates, and view pointer |
| `POINTER_DATA` | typedef | Byte pointer for generic pointer arithmetic |
| `CROSSPROD_TYPE` | typedef | Double-precision type for cross-product calculations to avoid overflow |

## Global / File-Static State
None.

## Key Functions / Methods

### `build_render_tree()`
- Signature: `void build_render_tree()`
- Purpose: Builds the visibility tree by casting rays from the viewpoint
- Inputs: Internal (uses `view`, `Nodes`, `PolygonQueue` member state)
- Outputs/Return: None; populates `Nodes` tree and clipping vectors
- Side effects: Fills `Nodes`, initializes `PolygonQueue`, populates `EndpointClips` and `LineClips`
- Calls: Implementation in corresponding .cpp file; likely calls `cast_render_ray()` for each screen boundary endpoint
- Notes: Core visibility algorithm; executed per frame to determine renderable polygons

### `cast_render_ray()`
- Signature: `void cast_render_ray(long_vector2d *_vector, short endpoint_index, int ParentIndex, short bias)`
- Purpose: Casts a ray through the polygon graph to determine visibility along a screen edge
- Inputs: `_vector` = ray direction (long vector for precision), `endpoint_index` = screen endpoint, `ParentIndex` = parent node index (not pointer), `bias` = direction preference
- Outputs/Return: None; adds `node_data` entries to `Nodes` vector
- Side effects: Modifies `Nodes` and clipping data; may recursively expand tree
- Calls: `decide_where_vertex_leads()`, recursive self-calls
- Notes: Uses index (not pointer) for parent node to avoid stale-pointer bugs when vector reallocates; fixes bug from original render.c

### `next_polygon_along_line()`
- Signature: `uint16 next_polygon_along_line(short *polygon_index, world_point2d *origin, long_vector2d *_vector, short *clipping_endpoint_index, short *clipping_line_index, short bias)`
- Purpose: Traverses the map's polygon adjacency graph along a ray direction
- Inputs: `polygon_index` = starting polygon (input/output), `origin` = ray start, `_vector` = ray direction, `bias` = traversal preference
- Outputs/Return: Status flags; output parameters contain next polygon and clipping indices
- Side effects: None (read-only graph traversal)
- Calls: Not visible; walks polygon sides/lines
- Notes: Implements state machine (`_looking_for_first_nonzero_vertex`, `_looking_clockwise_for_right_vertex`, etc.)

### `decide_where_vertex_leads()`
- Signature: `uint16 decide_where_vertex_leads(short *polygon_index, short *line_index, short *side_index, short endpoint_index_in_polygon_list, world_point2d *origin, long_vector2d *_vector, uint16 clip_flags, short bias)`
- Purpose: Determines which adjacent polygon a ray from a vertex leads into
- Inputs: Current polygon, vertex index, ray origin/direction, screen clipping flags, bias
- Outputs/Return: Flags indicating success; output parameters contain next polygon, line, and side
- Side effects: None
- Calls: Not visible
- Notes: Handles screen boundary clipping during traversal

### `calculate_endpoint_clipping_information()` / `calculate_line_clipping_information()`
- Signature: `short calculate_endpoint_clipping_information(short endpoint_index, uint16 clip_flags)` / `void calculate_line_clipping_information(short line_index, uint16 clip_flags)`
- Purpose: Create and cache clipping data for endpoints and lines that intersect screen boundaries
- Inputs: Map element index, clipping flags (which screen edges clip it)
- Outputs/Return: Endpoint version returns index into `EndpointClips`; line version returns void
- Side effects: Append to `EndpointClips` / `LineClips`; update `line_clip_indexes` translation table
- Calls: Not visible
- Notes: Lazy caching to avoid recalculating clipping for repeated elements

### `Resize()`
- Signature: `void Resize(size_t NumEndpoints, size_t NumLines)`
- Purpose: Pre-allocate memory based on map size
- Inputs: Estimated endpoint and line counts from map
- Outputs/Return: None
- Side effects: Resizes `endpoint_x_coordinates`, `EndpointClips`, `LineClips` vectors
- Calls: Not visible
- Notes: Mentioned as "lazy" ΓÇö likely allocates but does not initialize

**Trivial helpers (2ΓÇô3 bullets):**
- `PUSH_POLYGON_INDEX()` ΓÇö enqueues polygon index onto work queue
- `initialize_polygon_queue()`, `initialize_render_tree()`, `initialize_clip_data()` ΓÇö reset/prepare internal state; called by constructor
- `ResetEndpointClips()`, `ResetLineClips()` ΓÇö clear clipping data arrays between frames

## Control Flow Notes
This class is invoked once per frame during the visibility pass of rendering:
1. **Frame setup**: `build_render_tree()` is called with a populated `view` pointer (viewpoint and camera parameters)
2. **Root initialization**: Starting polygon determined from `view->origin_polygon_index`; `cast_render_ray()` called for each screen boundary endpoint to seed the tree
3. **Recursive traversal**: Each ray casts through adjacent polygons via `next_polygon_along_line()` and `decide_where_vertex_leads()`; visible polygons added to `Nodes` tree
4. **Clipping accumulation**: As rays cross screen edges, `calculate_endpoint_clipping_information()` and `calculate_line_clipping_information()` populate clipping data
5. **Output**: Final `Nodes` tree and clipping vectors consumed by rendering pass to determine what to draw

## External Dependencies
- **Includes**: `<vector>` (STL), `world.h`, `render.h`
- **From `world.h`**: `world_point2d`, `long_vector2d` (extended precision 2D vectors to avoid overflow on long distances), world coordinate system macros (`WORLD_ONE`, `NORMALIZE_ANGLE`, etc.)
- **From `render.h`**: `view_data` (viewpoint, viewport dimensions, camera orientation, field-of-view)
- **Defined elsewhere**: All function implementations (in RenderVisTree.cpp); map/polygon definitions (from game world)
