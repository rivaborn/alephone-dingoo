# Source_Files/RenderMain/NewRenderVisTree.cpp

## File Purpose
Implements the `NewVisTree` class, which builds a hierarchical visibility tree for a 3D game engine. The visibility tree determines which polygons are visible from the player's viewpoint, handling portal transitions and visibility culling through a queue-based algorithm that processes visible surfaces in depth order.

## Core Responsibilities
- Build visibility tree by recursively queuing visible polygons through transparent sides
- Transform map geometry into portal-local coordinate spaces using per-portal viewing parameters
- Maintain translation tables mapping world geometry indices to viewer-space indices
- Calculate screen-space clipping windows for each visible polygon
- Flatten the tree into a sorted, back-to-front list of renderable nodes
- Handle portal-within-portal viewing (nested portal views)
- Validate tree structure during debug builds

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `portal_view_data` | struct | Represents a viewpoint through a portal; holds transformations, translation tables, and view parameters for geometry within that portal |
| `render_node_data` | struct | A node in the visibility tree; represents one visible polygon with clipping windows and parent/child/sibling links |
| `sorted_node_data` | struct | A render node paired with a sort order and object indices for back-to-front rendering |
| `translated_endpoint_data` | struct | An endpoint transformed to viewer space, with screen x-coordinate and viewer-space vector |
| `clipping_window_data` | struct | A rectangular clipping region in screen space with bounding vectors for sub-pixel accuracy |
| `line_render_data` | struct (internal) | Queue entry: line, side, target polygon, and parent node index for processing |
| `line_clip_data` | struct (internal) | Cached clipping data for a line (left/right endpoints, ceiling/floor heights in view space) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `portal_view_data::memBlock` | `uint16*` | static | Shared memory pool for all portal view translation tables across all views in a frame |
| `portal_view_data::memBlockAlloced` | `int` | static | Current allocation offset in memBlock |
| `portal_view_data::memBlockSize` | `int` | static | Total size of memBlock |

## Key Functions / Methods

### portal_view_data::setup
- **Signature:** `bool setup(portal_reference portalRef, NewVisTree *visTree, uint16 parent_view_index)`
- **Purpose:** Initialize a portal view with coordinate transforms, allocate translation tables, and set viewing parameters for geometry behind a portal.
- **Inputs:** 
  - `portalRef`: reference to the portal being viewed through
  - `visTree`: parent visibility tree (for global view and portal list access)
  - `parent_view_index`: index of parent portal view (UNONE for root)
- **Outputs/Return:** `true` if setup succeeded; `false` if translation tables would exceed memBlock size
- **Side effects:** Allocates from static `memBlock`; sets `endpoint_translation_table`, `line_translation_table`, `polygon_node_table`; calculates origin, yaw, pitch, roll, and mirror flags
- **Calls:** `normalize_angle()`, `cosine_table[]`/`sine_table[]` lookups, `transform_long_point2d()`
- **Notes:** Z-mirrors are composed (XOR'd) through portal chains. Returns `false` gracefully if MAXIMUM_PORTAL_VIEWS (30) limit exceeded.

### NewVisTree::build_render_tree
- **Signature:** `void build_render_tree()`
- **Purpose:** Main entry point; constructs the entire visibility tree from scratch by queue-processing visible polygons.
- **Inputs:** None (uses member `view` pointer)
- **Outputs/Return:** None (populates `Nodes`, `SortedNodes`, `Endpoints`, `LineClips`, `ClippingWindows`, `PortalViews`)
- **Side effects:** Clears all lists at start; uses performance timers `gUsageTimers.visTree` and `gUsageTimers.visFlat`; calls `check_tree_integrity()` multiple times for validation
- **Calls:** `get_portal_view()`, `get_real_origin()`, `new_node()`, `new_endpoint()`, `get_clip_line()`, `clip_window_to_line()`, `LineQueue.push/pop()`, `flatten_render_tree()`, `rebuild_polygon_lookup()`
- **Notes:** Uses a goto to jump into the main loop for the root node (intentional design). The queue-based outer loop processes lines; each line creates a child node. Complex cross-product checks reject backfaces and handle degenerate (zero-cross-product) lines. Includes ancestor-chain check to prevent self-referential nodes (#183 bug workaround).

### NewVisTree::get_real_origin
- **Signature:** `polygon_reference get_real_origin(portal_view_data *portalView)`
- **Purpose:** Determine which polygon contains the viewer in a portal's local space (handles edge cases where viewer is on a polygon boundary).
- **Inputs:** `portalView`: portal view to search
- **Outputs/Return:** polygon index (may differ from view->origin_polygon_index if viewer sits on a portal line)
- **Side effects:** Calls `calculate_clip_endpoint()` to transform endpoints; may modify line endpoint ordering for zero-cross-product lines
- **Calls:** `polygon_reference()`, `line_reference()`, `calculate_clip_endpoint()`, `calculate_clip_line()`
- **Notes:** Uses cross-product and dot-product tests to check if viewer crosses into adjacent polygon. Handles symmetric ("Invisible Polygon Line Barrier of Illogical Infinity") by picking a consistent side for zero-cross-product lines.

### NewVisTree::calculate_clip_endpoint
- **Signature:** `int16 calculate_clip_endpoint(endpoint_reference map_index, portal_view_data *portal_view)`
- **Purpose:** Transform a map endpoint into viewer space, calculate screen x-coordinate, and cache the result in the portal view's translation table.
- **Inputs:** 
  - `map_index`: world-space endpoint index
  - `portal_view`: portal view context for transformation
- **Outputs/Return:** index in `Endpoints` list
- **Side effects:** Pushes to `Endpoints` vector; updates `portal_view->endpoint_translation_table`; calls `transform_long_point2d()`
- **Calls:** `transform_long_point2d()`, `PIN()`
- **Notes:** Returns `x = -1` if endpoint is outside the left/right cone edges. Uses fixed-point division and half-screen width for screen coordinate calculation.

### NewVisTree::calculate_clip_line
- **Signature:** `int16 calculate_clip_line(line_reference map_index, portal_view_data *portal_view)`
- **Purpose:** Compute top/bottom clip distances for a line and determine left/right endpoint ordering in viewer space.
- **Inputs:** 
  - `map_index`: world-space line index
  - `portal_view`: portal view context
- **Outputs/Return:** index in `LineClips` list
- **Side effects:** Inserts into `LineClips` vector; ensures endpoints are transformed; updates `portal_view->line_translation_table`
- **Calls:** `calculate_clip_endpoint()`, `line->endpoint_0()`, `line->endpoint_1()`
- **Notes:** Orders endpoints by cross-product sign. Top/bottom distances computed as line's world-space ceiling/floor minus portal origin z.

### NewVisTree::clip_window_to_line
- **Signature:** `bool clip_window_to_line(clipping_window_data *window, uint16 line, uint16 side_flags)`
- **Purpose:** Narrow a clipping window by intersecting it with a line's vertical boundaries (ceiling/floor and left/right edges).
- **Inputs:** 
  - `window`: clipping rectangle to modify
  - `line`: index in `LineClips`
  - `side_flags`: flags indicating whether side clips top/bottom aggressively
- **Outputs/Return:** `true` if window becomes empty after clipping; `false` otherwise
- **Side effects:** Modifies `window` members (x0, x1, y0, y1, vectors)
- **Calls:** `find_line_vector_intersection()`
- **Notes:** Uses "tight" and "loose" clipping strategies for top/bottom. Tight clipping uses the line's actual height; loose clipping uses the view cone edges. Complex logic selects which clip vector to use based on line endpoint distances and orientation.

### NewVisTree::flatten_render_tree
- **Signature:** `void flatten_render_tree()`
- **Purpose:** Traverse the visibility tree depth-first, output render nodes in back-to-front order, and merge clipping windows within each polygon.
- **Inputs:** None (uses `Nodes`, `root_node`)
- **Outputs/Return:** None (populates `SortedNodes`, extends `ClippingWindows`)
- **Side effects:** Unlinks nodes from tree structure during traversal (sets `parent` to UNONE); calls `qsort_windows()` to sort windows
- **Calls:** `check_tree_integrity()`, `qsort_windows()`, `sort_windows_y()`
- **Notes:** Handles two cases: single window (fast path, just delink) and multiple windows (sort, merge, insert remaining into ClippingWindows list). Merges windows if they share the same x-extent or y-extent and are contiguous.

### NewVisTree::rebuild_polygon_lookup
- **Signature:** `void rebuild_polygon_lookup()`
- **Purpose:** Reconstruct the polygonΓåÆnode index tables in each portal view after flattening.
- **Inputs:** None (uses `PortalViews`, `SortedNodes`)
- **Outputs/Return:** None (updates `polygon_node_table` in each portal view)
- **Side effects:** Memsets all polygon_node_tables to -1 (UNONE)
- **Calls:** None
- **Notes:** **Bug**: function has syntax error (`int i=0` missing semicolon), and iteration bounds check is wrong (`i<PortalViews.size` should be `i<PortalViews.size()`).

### NewVisTree::get_portal_view
- **Signature:** `uint16 get_portal_view(portal_reference portal, uint16 parent_view)`
- **Purpose:** Retrieve an existing portal view, or create and cache a new one if not found.
- **Inputs:** 
  - `portal`: portal to view through
  - `parent_view`: parent portal view index (UNONE for root)
- **Outputs/Return:** index in `PortalViews` (or UNONE if setup failed)
- **Side effects:** May insert into `PortalViews` vector; calls `portal_view_data::setup()`
- **Calls:** `portal_view_data::setup()`
- **Notes:** Linear search through existing portal views; acceptable for Γëñ ~10 portal views per frame.

### check_tree_integrity (debug stub)
- **Signature:** `void check_tree_integrity()`
- **Purpose:** Validate the visibility tree structure (parent/child/sibling links, node ownership, ancestry).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** None (early return disables checks; asserts on failure if enabled)
- **Calls:** `polygon_reference()`, `dassert()`, `assert()`
- **Notes:** Disabled by early `return` statement. When enabled, performs O(n┬▓) validation: climbs each node's parent chain to root, checks polygon uniqueness in paths, and validates that parents link back to children.

**Helper functions** (trivial, summarized):
- `qsort_windows()`: Quicksort partitioning by x0, with fallback to `sort_windows_y()` for ties
- `sort_windows_y()`: O(n┬▓) selection sort for small window counts

## Control Flow Notes
The visibility tree is built in phases:
1. **Initialization** (`build_render_tree` start): Clear lists, create root portal view and root node.
2. **Queue-based tree expansion** (outer `while` loop): Pop a line from `LineQueue`, create a child node, clip it, set up visibility. Recursively queue visible lines in new polygon via `process_node` label.
3. **Depth-first traversal** (`flatten_render_tree`): Visit nodes in tree order (children before siblings), aggregate clipping windows per polygon, merge windows, insert overflows into global list.
4. **Lookup rebuild** (`rebuild_polygon_lookup`): Restore polygonΓåÆnode mapping for next frame.

Frame/render integration: tree is rebuilt each frame. Cached `portal_view_data::memBlock` is reused; allocation resets when parent_view_index == UNONE (root portal).

## External Dependencies
- **map.h**: `polygon_data`, `line_data`, `endpoint_data`, `side_data`, `portal_data`; reference types (`polygon_reference`, `line_reference`, etc.); global lists (`EndpointList`, `LineList`, `SideList`, `PolygonList`)
- **world.h**: `long_vector2d`, `long_point2d`, `world_point3d`, `world_distance`; angle type and trig tables (`normalize_angle`, `cosine_table`, `sine_table`)
- **render.h**: `view_data` type (not fully defined here)
- **WorkQueue.h**: `WorkQueue<T>` template
- **cseries.h**: Macros (`WRAP_HIGH`, `PIN`, `TEST_FLAG`); `NewPtr()`, `bad_alloc`
- **readout_data.h**: `gUsageTimers` performance tracking struct
- **Not defined here**: `transform_long_point2d()`, `find_line_vector_intersection()` (defined elsewhere in engine)
