# Source_Files/RenderMain/RenderPlaceObjs.h

## File Purpose

Defines a class for placing game inhabitants (objects/actors) in appropriate rendering order during the frame render. Works in conjunction with visibility and polygon sorting systems to determine which objects should be rendered and in what depth order. Part of the rendering pipeline after visibility determination and polygon sorting.

## Core Responsibilities

- Build and maintain a list of render objects to be drawn
- Create individual render objects with world position, lighting intensity, and opacity
- Sort render objects into a spatial tree structure for depth ordering
- Determine visible base nodes from an object's position
- Compute per-object clipping windows for occlusion against visible polygons
- Rescale shape geometry for screen projection
- Integrate with visibility tree and sorted polygon systems

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `render_object_data` | struct | Encapsulates an object instance: its node location, clipping windows, linked-list chain, screen rectangle, and y-media value |
| `sorted_node_data` | struct (from RenderSortPoly.h) | Sorted polygon node with associated interior/exterior render objects and clipping windows |
| `shape_information_data` | struct (from interface.h) | Shape rendering metadata (flags, intensity, world bounds, hotpoint) |
| `view_data` | struct (from render.h) | Current camera state, FOV, screen dimensions, projection parameters |

## Global / File-Static State

None.

## Key Functions / Methods

### RenderPlaceObjsClass() [constructor]
- **Signature:** `RenderPlaceObjsClass()`
- **Purpose:** Initialize the render object placement manager
- **Inputs:** None
- **Outputs/Return:** Object instance
- **Side effects:** Initializes the render objects list
- **Calls:** `initialize_render_object_list()`
- **Notes:** Must be paired with pointers to `view`, `RVPtr`, `RenderVisTreeClass`, and `RSPtr` set after construction

### build_render_object_list() [public]
- **Signature:** `void build_render_object_list()`
- **Purpose:** Main entry point; iterates inhabitants and builds sorted render object list
- **Inputs:** Uses member pointers: `view`, `RVPtr`, `RSPtr`
- **Outputs/Return:** Populates `RenderObjects` vector
- **Side effects:** Clears and rebuilds entire render object list
- **Calls:** `initialize_render_object_list()`, `build_render_object()`, `sort_render_object_into_tree()`
- **Notes:** Called once per frame before rendering; coordinates with visibility and polygon sort systems

### build_render_object() [private]
- **Signature:** `render_object_data *build_render_object(long_point3d *origin, _fixed floor_intensity, _fixed ceiling_intensity, sorted_node_data **base_nodes, short *base_node_count, short object_index, float Opacity)`
- **Purpose:** Create a single render object instance with calculated clipping and bounds
- **Inputs:** Object origin (long 3D point), floor/ceiling light intensity, visible base nodes, object index, opacity
- **Outputs/Return:** Pointer to newly allocated `render_object_data`
- **Side effects:** Allocates object, modifies `RenderObjects` vector, calls clipping computation
- **Calls:** `build_aggregate_render_object_clipping_window()`, `rescale_shape_information()`
- **Notes:** Opacity parameter used for transparency; clipping windows computed relative to base nodes

### sort_render_object_into_tree() [private]
- **Signature:** `void sort_render_object_into_tree(render_object_data *new_render_object, sorted_node_data **base_nodes, short base_node_count)`
- **Purpose:** Insert render object into spatial tree for depth ordering
- **Inputs:** Render object, array of visible base nodes, node count
- **Outputs/Return:** Void; modifies object chain links
- **Side effects:** Modifies `sorted_node_data` interior/exterior object chains
- **Calls:** Direct tree manipulation (not shown)
- **Notes:** Determines whether object goes into interior or exterior chain of nodes

### build_base_node_list() [private]
- **Signature:** `short build_base_node_list(short origin_polygon_index, world_point3d *origin, world_distance left_distance, world_distance right_distance, sorted_node_data **base_nodes)`
- **Purpose:** Determine which sorted nodes are visible from object's origin point
- **Inputs:** Polygon index containing object, world position, left/right distance bounds
- **Outputs/Return:** Count of visible base nodes; populates `base_nodes` array
- **Side effects:** Queries visibility tree
- **Calls:** `RVPtr` visibility queries
- **Notes:** Bounds checking uses world distance (fixed-point values with WORLD_FRACTIONAL_BITS)

### build_aggregate_render_object_clipping_window() [private]
- **Signature:** `void build_aggregate_render_object_clipping_window(render_object_data *render_object, sorted_node_data **base_nodes, short base_node_count)`
- **Purpose:** Compute screen-space clipping window by aggregating visible node clipping
- **Inputs:** Render object, visible base nodes
- **Outputs/Return:** Void; populates `clipping_window_data` in render object
- **Side effects:** Modifies object's clipping windows; may call vertical clip calculation
- **Calls:** `calculate_vertical_clip_data()` (from RenderSortPolyClass)
- **Notes:** Determines which screen regions can draw this object without occlusion

### rescale_shape_information() [private]
- **Signature:** `shape_information_data *rescale_shape_information(shape_information_data *unscaled, shape_information_data *scaled, uint16 flags)`
- **Purpose:** Adapt shape geometry bounds and intensity for screen projection
- **Inputs:** Original shape data, destination buffer, flags (mirror/keypoint bits)
- **Outputs/Return:** Pointer to scaled shape data
- **Side effects:** Modifies `scaled` structure with projected values
- **Calls:** None visible
- **Notes:** Respects `_X_MIRRORED_BIT`, `_Y_MIRRORED_BIT`, `_KEYPOINT_OBSCURED_BIT`

## Control Flow Notes

**Position in render pipeline:**
1. **Visibility Tree** (`RenderVisTreeClass`) ΓåÆ determines visible polygons/nodes
2. **Polygon Sort** (`RenderSortPolyClass`) ΓåÆ orders polygons by depth
3. **Place Objects** (`RenderPlaceObjsClass`) ΓåÆ places inhabitants within sorted structure
4. (Implicit) Rendering loop draws sorted objects with their clipping windows

`build_render_object_list()` is the frame entry point, called once per rendered frame to regenerate the inhabitant list based on current view and visibility state.

## External Dependencies

- `<vector>`: STL dynamic array for `RenderObjects`
- `"world.h"`: `long_point3d`, `world_point3d`, `world_distance`, `_fixed` types
- `"interface.h"`: `shape_information_data`, rendering constants/flags
- `"render.h"`: `view_data` structure, rendering macros
- `"RenderSortPoly.h"`: `sorted_node_data`, `clipping_window_data`, `RenderSortPolyClass`, `RenderVisTreeClass` (forward)
