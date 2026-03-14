# Source_Files/RenderMain/NewRenderVisTree.h

## File Purpose
Defines a visibility tree class for portal-based rendering culling and clipping. Manages a hierarchical render tree of polygons, clipping windows, and portal transitions to optimize visibility determination and determine what surfaces are rendered each frame.

## Core Responsibilities
- Build and maintain hierarchical render tree from world geometry
- Translate world-space coordinates into rendering-space through portal chains
- Manage clipping windows for visibility culling at elevation lines and portals
- Track unique endpoints, lines, and polygons across portal views
- Sort render nodes back-to-front for correct draw order
- Cache computed clip data to avoid redundant calculations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `clipping_window_data` | struct | Screen-space clipping rectangle with viewport edges; extends `rasterize_window` |
| `render_node_data` | struct | A polygon in the render tree with hierarchy, clipping, and portal context |
| `sorted_node_data` | struct | A render node paired with interior/exterior object indices and sort order |
| `portal_view_data` | struct | A camera view through a specific portal with transformed coordinates and lookup tables |
| `translated_endpoint_data` | struct | An endpoint transformed to screen-space with viewer-space vector |
| `line_render_data` | struct (nested) | Queued line data: line, side, target polygon, parent node |
| `line_clip_data` | struct (nested) | Clipping bounds for a line: endpoints and vertical range |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `portal_view_data::memBlock` | `uint16*` | static (class) | Memory pool for portal view allocation |
| `portal_view_data::memBlockAlloced` | `int` | static (class) | Current allocation count |
| `portal_view_data::memBlockSize` | `int` | static (class) | Total pool size |

## Key Functions / Methods

### NewVisTree::build_render_tree
- Signature: `void build_render_tree()`
- Purpose: Build the complete visibility tree by traversing portal chains and computing clipping geometry
- Inputs: None (uses `view` member)
- Outputs/Return: Populates `Nodes`, `PortalViews`, `Endpoints`, `ClippingWindows`, `SortedNodes` vectors
- Side effects: Modifies all member vectors; caches clipping data
- Calls (visible here): `get_real_origin()`, `flatten_render_tree()`, `check_tree_integrity()`
- Notes: Core algorithm; likely calls portal traversal and clipping logic from implementation file

### NewVisTree::calculate_clip_endpoint
- Signature: `int16 calculate_clip_endpoint(endpoint_reference map_index, portal_view_data *portal_view)`
- Purpose: Transform an endpoint into portal view's coordinate system and cache result
- Inputs: Map endpoint reference, target portal view
- Outputs/Return: Index into `Endpoints` vector (or translation table for reuse)
- Side effects: Appends to `Endpoints` if new; updates portal view's translation table
- Calls: `new_endpoint()`

### NewVisTree::calculate_clip_line
- Signature: `int16 calculate_clip_line(line_reference map_index, portal_view_data *portal_view)`
- Purpose: Compute clipping bounds for a line in portal view space
- Inputs: Map line reference, target portal view
- Outputs/Return: Index into `LineClips` vector
- Side effects: Caches result in portal view's line translation table
- Calls: None visible

### NewVisTree::clip_window_to_line
- Signature: `bool clip_window_to_line(clipping_window_data *window, uint16 line, uint16 side_flags)`
- Purpose: Intersect a clipping window against a line's clipping bounds
- Inputs: Window to clip, line index, side flags (left/right/up/down indicators)
- Outputs/Return: `true` if window becomes empty; window modified in-place
- Side effects: Modifies clipping window geometry
- Notes: Implements 2D windowΓÇôline intersection

### get_clip_endpoint / get_clip_line
- Signature: `uint16 get_clip_endpoint(endpoint_reference map_index, uint16 portal_index)` / similar for lines
- Purpose: Accessor pattern: return cached clip data or compute and cache on-demand
- Inputs: Map reference and portal view index
- Outputs/Return: Local index into `Endpoints` / `LineClips`
- Side effects: May call `calculate_*` for cache miss
- Notes: Avoids redundant transformations when same world element appears in same portal view

### portal_view_data::setup
- Signature: `bool setup(portal_reference portal, NewVisTree *visTree, uint16 parent_view_index)`
- Purpose: Initialize a portal view with coordinate transformation and lookup tables
- Inputs: Portal ID, parent tree, parent view index
- Outputs/Return: `true` on success
- Side effects: Allocates memory for translation tables; computes yaw/pitch/roll and origin
- Notes: Called once per unique portal transition during tree build

### Factory / Accessor Methods
- `new_node()`, `new_endpoint()`, `new_window()`: Append to respective vectors and return index
- `get_node()`, `get_portal_view()`, `get_window()`, `get_clip_endpoint()`, `get_clip_line()`: Safe accessor with assertions

### Cleanup / Validation
- `flatten_render_tree()`: Post-process tree after construction (likely removes empty nodes or sorts)
- `rebuild_polygon_lookup()`: Update `polygon_node_table` mappings
- `check_tree_integrity()`: Debug validation
- Trivial helpers: `whoops_pop_node()` ΓÇö rollback a failed node allocation

## Control Flow Notes
**Initialization phase:**
- Constructor `NewVisTree()` allocates member vectors
- `build_render_tree()` walks portal graph, creating render nodes and clipping windows
- `get_real_origin()` resolves the camera's starting polygon through portal chain
- Endpoints and lines are transformed and cached on-demand during traversal
- `flatten_render_tree()` and `check_tree_integrity()` finalize and validate

**Frame rendering:**
- The built tree is queried to determine visibility and clipping for each polygon
- `SortedNodes` provides back-to-front draw order

## External Dependencies
- **world.h**: `angle`, `world_distance`, `world_point3d`, `long_vector2d` (coordinate and rotation types)
- **render.h**: `view_data` (camera parameters), `polygon_reference`, `endpoint_reference`, `side_reference`, `line_reference`, `portal_reference` (opaque handles to world geometry)
- **WorkQueue.h**: Template FIFO queue for `line_render_data` batching
- **Standard library**: `<vector>` (dynamic arrays)
- **Undefined here (from bundled headers)**: `rasterize_window` (base class), world/render function definitions
