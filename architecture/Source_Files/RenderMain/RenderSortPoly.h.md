# Source_Files/RenderMain/RenderSortPoly.h

## File Purpose
Defines a class for sorting game world polygons into depth order during rendering. Works from visibility tree data to organize polygons with their associated render objects and clipping information for the renderer.

## Core Responsibilities
- Maintains mapping from map polygon indices to sorted render nodes
- Builds and stores clipping window data for polygon rendering
- Accumulates endpoint and line clipping information during rendering
- Resizes internal structures as polygon count changes
- Orchestrates the polygon depth-sorting process for a given view

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `sorted_node_data` | struct | Associates a polygon with its interior/exterior render objects and clipping windows |
| `RenderSortPolyClass` | class | Main sorting container; holds sorted nodes, clipping data, and view context |

## Global / File-Static State
None.

## Key Functions / Methods

### RenderSortPolyClass (constructor)
- Signature: `RenderSortPolyClass()`
- Purpose: Initializes the sorting class
- Inputs: None
- Outputs/Return: Constructed instance
- Side effects: Allocates default state for vectors
- Calls: (Implicit vector construction)
- Notes: No explicit implementation shown; likely does basic initialization

### sort_render_tree
- Signature: `void sort_render_tree()`
- Purpose: Main entry point; sorts polygons from the visibility tree into depth order
- Inputs: Uses internal `view`, `RVPtr` (visibility tree), and `SortedNodes` vector
- Outputs/Return: Populates `SortedNodes` and updates `polygon_index_to_sorted_node` mapping
- Side effects: Modifies `SortedNodes` size and contents; updates both accumulation vectors for clipping
- Calls: (Implementation in .cpp not shown; likely calls `initialize_sorted_render_tree()` and private helpers)
- Notes: Updates accumulation vectors during construction

### Resize
- Signature: `void Resize(size_t NumPolygons)`
- Purpose: Lazily resizes all internal vectors to accommodate a given polygon count
- Inputs: `NumPolygons` ΓÇö expected maximum polygon count
- Outputs/Return: None
- Side effects: Grows `polygon_index_to_sorted_node`, `SortedNodes`, `AccumulatedEndpointClips`, `AccumulatedLineClips` to fit NumPolygons
- Calls: (Implicit vector resize operations)
- Notes: Lazy resizing means actual allocation may defer until needed

### initialize_sorted_render_tree (private)
- Signature: `void initialize_sorted_render_tree()`
- Purpose: Internal setup; prepares sorted node structure before sorting
- Inputs: Internal state (view, visibility tree)
- Outputs/Return: None
- Side effects: Modifies `SortedNodes` vector length and content
- Calls: (Implementation in .cpp not shown)
- Notes: Called during sorting; render objects not yet listed when run

### build_clipping_windows (private)
- Signature: `clipping_window_data *build_clipping_windows(node_data *ChainBegin)`
- Purpose: Constructs clipping window hierarchy starting from a node chain
- Inputs: `ChainBegin` ΓÇö head of a linked list of visibility tree nodes
- Outputs/Return: Pointer to constructed clipping window data
- Side effects: Populates `AccumulatedEndpointClips` and `AccumulatedLineClips`
- Calls: `calculate_vertical_clip_data()` (likely)
- Notes: Returns pointer to clipping window; uses accumulated clip vectors

### calculate_vertical_clip_data (private)
- Signature: `void calculate_vertical_clip_data(line_clip_data **accumulated_line_clips, size_t accumulated_line_clip_count, clipping_window_data *window, short x0, short x1)`
- Purpose: Computes vertical screen-space clipping bounds for a clipping window
- Inputs: Accumulated line clips array with count, target window, horizontal screen range (x0, x1)
- Outputs/Return: None
- Side effects: Modifies fields in `window`
- Calls: (Implementation not shown)
- Notes: Part of clipping computation pipeline

## Control Flow Notes
**Init/Frame Flow:**
- `Resize()` is called when polygon count changes (lazy allocation)
- `sort_render_tree()` is called per frame/view render
  - Calls `initialize_sorted_render_tree()` to reset state
  - Internally builds clipping windows via `build_clipping_windows()`
  - Accumulates clip data for screen-space clipping

**View Context:**
- Holds pointer to current `view_data` (camera/viewport state)
- References `RenderVisTreeClass` which provides visibility graph
- Produces sorted nodes ready for the final rendering pass

## External Dependencies
- **Includes:**  
  `<vector>` (STL), `world.h`, `render.h`, `RenderVisTree.h`
- **Defined Elsewhere:**
  - `view_data` ΓÇö rendering view/camera state (render.h)
  - `node_data` ΓÇö visibility tree node (RenderVisTree.h)
  - `clipping_window_data`, `endpoint_clip_data`, `line_clip_data` ΓÇö clipping structures (RenderVisTree.h)
  - `render_object_data` ΓÇö render object structure (defined elsewhere, used in sorted_node_data)
  - `RenderVisTreeClass` ΓÇö visibility tree builder (RenderVisTree.h)
