# Source_Files/RenderMain/RenderVisTree.cpp

## File Purpose
Implements the visibility tree builder for the Aleph One game renderer. Constructs a hierarchical tree of visible polygons from the player's viewpoint by ray-casting against map geometry, handling viewport clipping and elevation changes for correct rendering order.

## Core Responsibilities
- Build a visibility tree by breadth-first processing of polygons seen from the player's location
- Cast rays from the viewpoint to detect polygon transitions and determine visibility
- Manage screen-space clipping data (left/right/top/bottom edges and elevation changes)
- Track transformed endpoint coordinates and clipping endpoints/lines per node
- Handle long-distance calculations with overflow corrections
- Maintain a binary search tree of nodes sorted by polygon index for efficient traversal

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `node_data` | struct | Tree node representing a visible polygon; contains parent/sibling/children pointers and clipping data; includes polygon-sort (binary) tree pointers for rendering order |
| `endpoint_clip_data` | struct | Clipping window descriptor for screen-edge endpoints (left/right sides of screen) |
| `line_clip_data` | struct | Clipping window descriptor for elevation lines (top/bottom screen bounds); stores transformed screen-space y-coordinates |
| `clipping_window_data` | struct | Bounds of a clipping region; groups left/right and top/bottom vectors |

## Global / File-Static State
None (all state is member-owned by `RenderVisTreeClass`).

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Nodes` | `vector<node_data>` | instance | Visibility tree nodes; index 0 is root at player's origin polygon |
| `PolygonQueue` | `vector<short>` | instance | Work queue of polygon indices to process; `polygon_queue_size` tracks active count |
| `EndpointClips` | `vector<endpoint_clip_data>` | instance | Screen-edge clipping data; indices 0ΓÇô1 are left/right screen edges (constant) |
| `LineClips` | `vector<line_clip_data>` | instance | Elevation-line clipping data; index 0 is top/bottom screen (constant) |
| `ClippingWindows` | `vector<clipping_window_data>` | instance | Clipping window list (unused in this file; populated elsewhere) |
| `endpoint_x_coordinates` | `vector<short>` | instance | Screen-space x-coordinate per endpoint index (only valid if endpoint is transformed) |
| `line_clip_indexes` | `vector<size_t>` | instance | Map from map line index to line clip index; set when a line gets clip data |

## Key Functions / Methods

### build_render_tree
- Signature: `void build_render_tree()`
- Purpose: Main entry point; constructs the complete visibility tree from the player's viewpoint
- Inputs: None; uses `view` member (must be set by caller)
- Outputs/Return: Mutates `Nodes`, `PolygonQueue`, clipping data
- Side effects: Populates visibility tree, clipping buffers, marks polygons/endpoints/lines as visible via render flags
- Calls: `initialize_polygon_queue()`, `initialize_render_tree()`, `initialize_clip_data()`, `cast_render_ray()`, `PUSH_POLYGON_INDEX()`, `get_polygon_data()`, `get_endpoint_data()`, `transform_overflow_point2d()`, `short_to_long_2d()`, `overflow_short_to_long_2d()`
- Notes: Breadth-first; processes all queued polygons, expanding visibility as rays hit new polygons. Casts rays to left and right screen edges first, then to each endpoint.

### cast_render_ray
- Signature: `void cast_render_ray(long_vector2d *_vector, short endpoint_index, int ParentIndex, short bias)`
- Purpose: Recursively follow a ray from the parent polygon, building tree nodes as the ray crosses polygon boundaries
- Inputs:
  - `_vector`: direction vector (world space, long precision)
  - `endpoint_index`: target endpoint (may be NONE if splitting)
  - `ParentIndex`: index of parent node in `Nodes` vector
  - `bias`: clipping direction (_no_bias, _clockwise_bias, _counterclockwise_bias)
- Outputs/Return: None; populates tree nodes under parent
- Side effects: Appends nodes to `Nodes` vector (may trigger reallocation); updates pointer chains; marks lines/endpoints with clipping data; enqueues polygons
- Calls: `next_polygon_along_line()`, `INITIALIZE_NODE()`, `calculate_line_clipping_information()`, `calculate_endpoint_clipping_information()`, `decide_where_vertex_leads()`
- Notes: Must track `ParentIndex` (not a pointer) to avoid stale-pointer bugs when `Nodes` reallocates; re-syncs parent pointer after realloc. Splits rays at non-biased vertices by recursing with clockwise and counterclockwise biases.

### next_polygon_along_line (non-Dingoo version)
- Signature: `uint16 next_polygon_along_line(short *polygon_index, world_point2d *origin, long_vector2d *_vector, short *clipping_endpoint_index, short *clipping_line_index, short bias)`
- Purpose: Given a polygon and a ray, find the next polygon the ray crosses into and detect elevation clipping
- Inputs:
  - `*polygon_index`: current polygon
  - `*origin`: ray origin in world space
  - `*_vector`: ray direction vector (long precision)
  - `*clipping_endpoint_index`: endpoint we're targeting (may change to NONE if not hit)
  - `*clipping_line_index`: set if crossing an elevation line
  - `bias`: search direction (clockwise/counterclockwise)
- Outputs/Return: Updates `*polygon_index` to next polygon (or NONE if edge of map); returns clip flags (_clip_up, _clip_down, _split_render_ray)
- Side effects: Enqueues the current polygon; adds line/polygon to automap; marks sides visible
- Calls: `get_polygon_data()`, `get_endpoint_data()`, `decide_where_vertex_leads()`, `get_line_data()`, `ADD_POLYGON_TO_AUTOMAP()`, `PUSH_POLYGON_INDEX()`, `ADD_LINE_TO_AUTOMAP()`, `SET_RENDER_FLAG()`, `LINE_IS_TRANSPARENT()`
- Notes: State machine searches for polygon boundary crossings using cross-product tests. Two versions: high-precision (default) and low-precision (Dingoo build); differs in cross-product calculation. Handles collinear vertices by dispatching to `decide_where_vertex_leads()`.

### decide_where_vertex_leads
- Signature: `uint16 decide_where_vertex_leads(short *polygon_index, short *line_index, short *side_index, short endpoint_index_in_polygon_list, world_point2d *origin, long_vector2d *_vector, uint16 clip_flags, short bias)`
- Purpose: When a ray touches a vertex, determine which polygon is on the other side
- Inputs: Polygon, vertex index in that polygon, ray origin/direction, current clipping flags, bias
- Outputs/Return: Updates polygon/line/side to point to next polygon; returns updated clip flags
- Side effects: None
- Calls: `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`
- Notes: No-bias case sets `_split_render_ray` flag to request ray splitting.

### calculate_line_clipping_information
- Signature: `void calculate_line_clipping_information(short line_index, uint16 clip_flags)`
- Purpose: Compute screen-space clipping bounds for a line that has elevation change
- Inputs: Map line index, clipping flags (_clip_up and/or _clip_down)
- Outputs/Return: Appends entry to `LineClips`; updates `line_clip_indexes[line_index]`
- Side effects: Extends `LineClips` vector; sets `_line_has_clip_data` flag
- Calls: `get_line_data()`, `get_endpoint_data()`, `transform_overflow_point2d()`, `overflow_short_to_long_2d()`, `SET_RENDER_FLAG()`, `TEST_RENDER_FLAG()`
- Notes: Handles both screen x-boundaries and perspective y-transformation; pins to screen edges.

### calculate_endpoint_clipping_information
- Signature: `short calculate_endpoint_clipping_information(short endpoint_index, uint16 clip_flags)`
- Purpose: Compute clipping vector for an endpoint that clips against screen edge
- Inputs: Endpoint index, clip flags (_clip_left or _clip_right)
- Outputs/Return: Returns index in `EndpointClips` (or NONE if endpoint not transformed)
- Side effects: Extends `EndpointClips` vector; sets `_endpoint_has_clip_data` flag
- Calls: `TEST_RENDER_FLAG()`, `get_endpoint_data()`, `overflow_short_to_long_2d()`
- Notes: Only creates clip entry if endpoint was already transformed; validates that flag is set exactly once.

### initialize_render_tree
- Signature: `void initialize_render_tree()`
- Purpose: Create root node at player's current polygon
- Inputs: None; uses `view->origin_polygon_index`
- Outputs/Return: Clears `Nodes`, appends root node
- Side effects: None
- Calls: `INITIALIZE_NODE()`

### initialize_polygon_queue, initialize_clip_data
- Purpose: Reset queue and clipping data; set default screen-edge clips
- Side effects: Clears lists; pre-populates `EndpointClips` and `LineClips` with screen edge descriptors
- Calls: `ResetEndpointClips()`, `ResetLineClips()`, `short_to_long_2d()`

### PUSH_POLYGON_INDEX, INITIALIZE_NODE
- Purpose: Enqueue a polygon (if not already visited); initialize a node's fields
- Notes: `PUSH_POLYGON_INDEX` checks `_polygon_is_visible` flag; `INITIALIZE_NODE` zeroes clipping counts and sets pointer chain.

## Control Flow Notes
**Initialization phase**: Build tree starting from player's polygon. Cast rays to left/right screen edges first (with bias to split at edges). **Work phase**: Process polygon queue breadth-first. For each polygon, iterate endpoints; if not yet visited, transform and cast a ray to it. Rays cross boundaries and create child nodes; when rays exit the visible space or hit non-transparent geometry, they terminate.

**Visibility tree structure**: Nodes form a parentΓåÆchildren linked list (siblings chain children horizontally). Each node also links into a binary-sorted polygon-index tree for rendering order.

## External Dependencies
- **map.h**: `polygon_data`, `endpoint_data`, `line_data`, `side_data`; accessors `get_polygon_data()`, etc.; macros `TEST_RENDER_FLAG()`, `SET_RENDER_FLAG()`, `ENDPOINT_IS_TRANSPARENT()`, `LINE_IS_TRANSPARENT()`, etc.
- **render.h**: `view_data`, `world_to_screen_x`, `world_to_screen_y`, perspective/transform constants
- **world.h**: `world_point2d`, `world_point3d`, `long_vector2d`, `long_point2d`, transformation functions
- **cseries.h**: Utility macros (`PIN`, `WRAP_LOW`, `WRAP_HIGH`, `SWAP`), assertions, type aliases
- STL `<vector>`: Growable arrays replacing legacy resizable lists
