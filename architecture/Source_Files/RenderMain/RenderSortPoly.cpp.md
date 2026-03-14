# Source_Files/RenderMain/RenderSortPoly.cpp

## File Purpose
Implements polygon depth-sorting and clipping-window generation for the Marathon-like game engine's render pipeline. Decomposes a polygon visibility tree into depth-sorted order while accumulating screen-space clipping regions for each polygon.

## Core Responsibilities
- Decompose render tree via depth-first polygon traversal (removing leaves first)
- Build screen-space clipping windows for each visible polygon
- Accumulate endpoint and line clip data from parent polygon chains
- Calculate vertical clipping bounds (top/bottom) for rendering regions
- Manage sorted node data and polygon-to-sorted-node indexing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `sorted_node_data` | struct | Represents a sorted polygon with its clipping windows and render objects |
| `endpoint_clip_data` | struct (external) | Screen-space clipping point (left/right edges) |
| `line_clip_data` | struct (external) | Screen-space clipping line (top/bottom edges) |
| `clipping_window_data` | struct (external) | A rectangular clipping region with edge vectors and y-bounds |
| `node_data` | struct (external) | Render tree node with polygon index, parent/sibling/child pointers, clipping data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_looking_for_left_clip`, `_looking_for_right_clip`, `_building_clip_window` | enum | static | States for clipping window builder state machine |
| `MAXIMUM_SORTED_NODES` | #define (128) | file-static | Pre-allocated capacity hint for sorted nodes vector |
| `MAXIMUM_CLIPS_PER_NODE` | #define (64) | file-static | Pre-allocated capacity hint for clip accumulation vectors |

## Key Functions / Methods

### RenderSortPolyClass::RenderSortPolyClass
- **Signature:** `RenderSortPolyClass()`
- **Purpose:** Initialize the polygon sorter with reserved vector capacity.
- **Inputs:** None
- **Outputs/Return:** Constructed object with empty vectors
- **Side effects:** Pre-allocates vectors (`SortedNodes`, `AccumulatedEndpointClips`, `AccumulatedLineClips`) to avoid frequent reallocations
- **Calls:** `std::vector::reserve()`
- **Notes:** Sets `view` and `RVPtr` to NULL for safety; actual initialization via external calls to `sort_render_tree()`

### RenderSortPolyClass::Resize
- **Signature:** `void Resize(size_t NumPolygons)`
- **Purpose:** Resize the polygon-to-sorted-node lookup table when map dimensions change.
- **Inputs:** `NumPolygons` ΓÇö number of polygons in the map
- **Outputs/Return:** void
- **Side effects:** Resizes `polygon_index_to_sorted_node` vector
- **Calls:** `std::vector::resize()`
- **Notes:** Lazy resizing; called before sorting begins

### RenderSortPolyClass::initialize_sorted_render_tree
- **Signature:** `void initialize_sorted_render_tree()`
- **Purpose:** Clear the sorted nodes list before decomposition.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Clears `SortedNodes` vector
- **Calls:** `std::vector::clear()`
- **Notes:** Called at the start of `sort_render_tree()`

### RenderSortPolyClass::sort_render_tree
- **Signature:** `void sort_render_tree()`
- **Purpose:** Main tree decomposition: extract polygons in depth order (leaves first) via polynomial-sorted binary-search tree traversal.
- **Inputs:** Reads `RVPtr->Nodes` (render tree); reads `view` (for later screen bounds)
- **Outputs/Return:** void; populates `SortedNodes` in back-to-front depth order
- **Side effects:** 
  - Modifies render tree node pointers (removes nodes via `reference`/`siblings` chain updates)
  - Populates `polygon_index_to_sorted_node` lookup table
  - Calls `build_clipping_windows()` for each sorted polygon
- **Calls:**
  - `initialize_sorted_render_tree()`
  - `build_clipping_windows(node_data *)`
  - `get_polygon_data(short)`
- **Notes:**
  - Uses binary-search tree structure (`PS_Greater`, `PS_Less`, `PS_Shared` pointers) on polygon indices to find all nodes matching a polygon (aliases)
  - Leaf detection: a polygon is a leaf if no node with that polygon index has children
  - Destructively removes nodes from tree via linked-list manipulation (`reference`, `siblings`)
  - Processes sibling/parent chain after removing a leaf

### RenderSortPolyClass::build_clipping_windows
- **Signature:** `clipping_window_data *build_clipping_windows(node_data *ChainBegin)`
- **Purpose:** Build screen-space clipping windows for a polygon and its parent chain, handling both endpoint and line clips.
- **Inputs:** 
  - `ChainBegin` ΓÇö first node in the polygon-sharing chain (PS_Shared linked list)
- **Outputs/Return:** Pointer to first `clipping_window_data` in a linked list (via `next_window`), or NULL
- **Side effects:**
  - Clears and populates `AccumulatedEndpointClips` and `AccumulatedLineClips`
  - Allocates clipping windows into `RVPtr->ClippingWindows` vector
  - Updates pointers in all allocated windows if vector reallocates
  - Updates `polygon_index_to_sorted_node` references if vector reallocates
  - Calls `calculate_vertical_clip_data()` for each window
- **Calls:**
  - `get_polygon_data(short)`
  - `calculate_vertical_clip_data(line_clip_data **, size_t, clipping_window_data *, short, short)`
  - `memmove()` for sorted insertion
  - `std::vector::push_back()`, vector reallocate handlers
- **Notes:**
  - Walks polygon vertex endpoints to compute real screen bounds (`x0`, `x1`)
  - Accumulates clips from node and all ancestors (parent chain)
  - Sorts endpoint clips left-to-right; avoids duplicate line clips
  - Finite-state machine builds windows from pairs of left/right endpoint clips
  - Validates window bounds: `left_clip->x < right_clip->x`, within screen width

### RenderSortPolyClass::calculate_vertical_clip_data
- **Signature:** `void calculate_vertical_clip_data(line_clip_data **accumulated_line_clips, size_t accumulated_line_clip_count, clipping_window_data *window, short x0, short x1)`
- **Purpose:** Compute top and bottom clipping edges for a horizontal screen region.
- **Inputs:**
  - `accumulated_line_clips` ΓÇö array of pointers to line clip data
  - `accumulated_line_clip_count` ΓÇö number of clips
  - `window` ΓÇö clipping window to populate with top/bottom edges
  - `x0`, `x1` ΓÇö horizontal region bounds
- **Outputs/Return:** void; populates `window->top`, `window->y0`, `window->bottom`, `window->y1`
- **Side effects:** Writes to `clipping_window_data` structure
- **Calls:** None (inline logic only)
- **Notes:**
  - Scans line clips iteratively: at each x-position, finds highest top clip and lowest bottom clip
  - Advances x by segment endpoints (`locally_highest_line->x1`)
  - Asserts that a clip exists for every x in range (invariant: no gaps)
  - Handles `_clip_up` and `_clip_down` flags independently

## Control Flow Notes
Part of the **pre-render pipeline**: called after visibility tree construction (`RenderVisTree`), before object rendering. Produces back-to-front polygon order with per-polygon clipping windows that guard the rasterization stage. Fits into: visibility culling ΓåÆ tree decomposition (this file) ΓåÆ object collection ΓåÆ rendering.

## External Dependencies
- **Includes:** `cseries.h` (utility macros, types), `map.h` (polygon/line/endpoint data, `get_polygon_data()`), `RenderSortPoly.h` (class definition)
- **External symbols:** 
  - `view_data` (contains `screen_width`)
  - `node_data`, `polygon_data`, `endpoint_clip_data`, `line_clip_data`, `clipping_window_data` (defined elsewhere in render system)
  - `RenderVisTreeClass` (visibility tree; provides `Nodes`, `EndpointClips`, `LineClips`, `ClippingWindows`, `endpoint_x_coordinates` vectors)
  - Macros: `TEST_RENDER_FLAG()`, `vassert()`, `csprintf()`, `MAX()`, `MIN()`, `POINTER_CAST()`, `NONE`, `SHRT_MAX`, `SHRT_MIN`
