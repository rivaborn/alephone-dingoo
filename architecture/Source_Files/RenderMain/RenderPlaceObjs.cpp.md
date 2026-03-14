# Source_Files/RenderMain/RenderPlaceObjs.cpp

## File Purpose
Implements object-placement and depth-sorting for the Marathon game renderer. Places in-game objects (sprites, monsters, items) into sorted polygons for correct depth-based rendering, computing 2D screen projections, clipping windows, and handling 3D model bounding boxes.

## Core Responsibilities
- Build and maintain a growable list of render objects in depth order
- Compute 2D screen projections from 3D world coordinates
- Sort objects by depth relative to camera and polygon visibility tree
- Calculate clipping windows for objects spanning multiple polygons
- Handle parasitic objects (objects attached to hosts, like projectiles on monsters)
- Project 3D model bounding boxes and compute lighting directions for models
- Scale object sprite information based on size flags (enlarged/tiny objects)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `render_object_data` | struct | Single renderable object with node, clipping windows, depth info, and rectangle definition |
| `sorted_node_data` | struct (external) | Visibility tree node containing polygon and object lists |
| `shape_information_data` | struct (external) | Sprite/shape dimensions and flags (mirroring, light intensity) |
| `clipping_window_data` | struct (external) | Clipping region for a rendered object |
| `GLfloat BoundingBox[2][3]` | array | 3D bounding box (two 3D corner points) for model projection |
| `OGL_ModelData` | struct (external) | 3D model data with transformation and lighting info |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FindProjectedBoundingBox` | static function | file-static | Helper: projects 3D bounding box to 2D, computes miner's light direction |

## Key Functions / Methods

### RenderPlaceObjsClass::RenderPlaceObjsClass()
- **Signature:** `RenderPlaceObjsClass()`
- **Purpose:** Initialize render object placement system; reserve memory for render objects vector.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initializes `view`, `RVPtr`, `RSPtr` to NULL; reserves `MAXIMUM_RENDER_OBJECTS` slots in `RenderObjects` vector.
- **Calls:** `vector::reserve()`
- **Notes:** Idiot-proofing initializes pointers to NULL; vector resizing later will trigger pointer-fixup code.

### RenderPlaceObjsClass::build_render_object_list()
- **Signature:** `void build_render_object_list()`
- **Purpose:** Main entry point; walk sorted polygon visibility tree, collect all objects, compute projections, depth-sort, and build clipping windows.
- **Inputs:** Internal: `view`, `RSPtr` (sorted node tree), `RVPtr` (visibility tree).
- **Outputs/Return:** None; populates `RenderObjects` vector.
- **Side effects:** Clears and rebuilds `RenderObjects`; may reallocate vector memory.
- **Calls:** `initialize_render_object_list()`, `build_render_object()`, `build_aggregate_render_object_clipping_window()`, `sort_render_object_into_tree()`.
- **Notes:** Iterates polygons back-to-front; skips invisible objects; applies chase-cam opacity to player's own object.

### RenderPlaceObjsClass::build_render_object(long_point3d *origin, _fixed floor_intensity, _fixed ceiling_intensity, sorted_node_data **base_nodes, short *base_node_count, short object_index, float Opacity)
- **Signature:** `render_object_data *build_render_object(...)`
- **Purpose:** Build a single render object: compute screen projection, handle 3D models, set up transfer modes, and chain parasitic objects.
- **Inputs:**
  - `origin`: optional world position override (NULL = use object location)
  - `floor_intensity`, `ceiling_intensity`: polygon lighting
  - `object_index`: which object to render
  - `Opacity`: alpha for chase cam
- **Outputs/Return:** Pointer to newly created `render_object_data` in `RenderObjects` vector; NULL on failure (invisible, too far, no shape).
- **Side effects:** May reallocate `RenderObjects` vector (and updates all pointers in `sorted_node_data` and other `render_object_data`); recursively calls `build_render_object()` for parasitic objects.
- **Calls:** `get_object_data()`, `get_object_shape_and_transfer_mode()`, `extended_get_shape_information()`, `rescale_shape_information()`, `OGL_GetModelData()`, `FindProjectedBoundingBox()`, `build_base_node_list()`, `extended_get_shape_bitmap_and_shading_table()`, `instantiate_rectangle_transfer_mode()`.
- **Notes:** Handles overflow arithmetic for long-distance viewing; clamps projected coordinates to short range; computes liquid relative height for media-affected rendering; checks visibility flags and handles nonexistent shapes gracefully.

### RenderPlaceObjsClass::sort_render_object_into_tree(render_object_data *new_render_object, sorted_node_data **base_nodes, short base_node_count)
- **Signature:** `void sort_render_object_into_tree(...)`
- **Purpose:** Insert new object(s) into correct depth order within the visibility tree.
- **Inputs:**
  - `new_render_object`: object(s) to insert (may be linked via `next_object`)
  - `base_nodes`: array of visibility nodes this object overlaps
  - `base_node_count`: number of nodes
- **Outputs/Return:** None; modifies object list in `base_nodes[*]->exterior_objects` (or interior for deep objects).
- **Side effects:** Updates `next_object` pointers; updates `node` field in new objects.
- **Calls:** None to external functions; iterates `RenderObjects` and `SortedNodes` vectors.
- **Notes:** Finds deepest and shallowest intersecting objects; adjusts desired node based on their constraints; places new object between them in linked list.

### RenderPlaceObjsClass::build_base_node_list(short origin_polygon_index, world_point3d *origin, world_distance left_distance, world_distance right_distance, sorted_node_data **base_nodes)
- **Signature:** `short build_base_node_list(...)`
- **Purpose:** Find all visibility nodes that an object spans (by casting rays left and right from object center).
- **Inputs:**
  - `origin_polygon_index`: polygon containing object
  - `origin`: 3D object position
  - `left_distance`, `right_distance`: sprite extents perpendicular to view direction
- **Outputs/Return:** Count of nodes added to `base_nodes[]` array (at most `MAXIMUM_OBJECT_BASE_NODES`).
- **Side effects:** Populates `base_nodes` array.
- **Calls:** `get_polygon_data()`, `get_endpoint_data()`, `translate_point2d()` (via macro).
- **Notes:** Ray-cast algorithm searches for polygon transitions; clips based on floor/ceiling height to avoid vertical traversal errors; gives up and returns current count if trapped.

### RenderPlaceObjsClass::build_aggregate_render_object_clipping_window(render_object_data *render_object, sorted_node_data **base_nodes, short base_node_count)
- **Signature:** `void build_aggregate_render_object_clipping_window(...)`
- **Purpose:** Compute merged clipping window(s) from all overlapped visibility nodes.
- **Inputs:**
  - `render_object`: object to clip
  - `base_nodes`: nodes that object spans
  - `base_node_count`: count
- **Outputs/Return:** None; stores clipping window(s) in `render_object->clipping_windows`.
- **Side effects:** May reallocate `ClippingWindows` vector (updates all pointers in sorted nodes and render objects).
- **Calls:** `vector::size()`, `vector::push_back()` (for clipping window allocation).
- **Notes:** Trivial case (single node) reuses existing window; multi-node case merges windows by finding min/max Y and building non-overlapping X segments; handles NULL window pointers defensively.

### RenderPlaceObjsClass::rescale_shape_information(shape_information_data *unscaled, shape_information_data *scaled, uint16 flags)
- **Signature:** `shape_information_data *rescale_shape_information(...)`
- **Purpose:** Apply size scaling (enlarged 1.25x or tiny 0.5x) to sprite dimensions.
- **Inputs:**
  - `unscaled`: original shape dimensions
  - `flags`: object flags (includes `_object_is_enlarged`, `_object_is_tiny`)
- **Outputs/Return:** Pointer to `scaled` (or `unscaled` if no flags); NULL if `unscaled` is NULL.
- **Side effects:** Populates `scaled` struct.
- **Calls:** None.
- **Notes:** Scales 6 distance values (world_left, world_right, world_top, world_bottom, world_x0, world_y0); if no flags, returns unscaled pointer directly.

### FindProjectedBoundingBox(GLfloat BoundingBox[2][3], long_point3d& TransformedPosition, GLfloat Scale, short RelativeAngle, shape_information_data& ShapeInfo, short DepthType, int& Farthest, int& ProjDistance, int& DistanceRef, int& LightDepth, GLfloat *Direction)
- **Signature:** `static void FindProjectedBoundingBox(...)`
- **Purpose:** Project 3D model bounding box to 2D, compute sprite rectangle bounds, farthest/closest depths, and miner's light direction for player illumination.
- **Inputs:**
  - `BoundingBox`: 8-corner 3D box (2 min/max points ├ù 3 coords)
  - `TransformedPosition`: object position relative to camera
  - `Scale`: object scale multiplier (1.25 for enlarged, 0.5 for tiny)
  - `RelativeAngle`: object rotation relative to camera yaw
  - `DepthType`: depth reference (-1=closest, 0=center, +1=farthest)
- **Outputs/Return:** None; outputs via references: `Farthest`, `ProjDistance`, `DistanceRef`, `LightDepth`, `Direction[3]`.
- **Side effects:** None.
- **Calls:** `normalize_angle()`, `cosine_table[]`, `sine_table[]`, `isqrt()`.
- **Notes:** Expands bounding box to 8 corners using 2D rotation (yaw) and scaling; clips X to avoid division by zero; computes min/max projected Y and Z; uses inverse square root for direction normalization.

---

## Control Flow Notes

**Initialization phase:** `RenderPlaceObjsClass` constructor reserves memory.

**Frame render phase:**
1. `build_render_object_list()` is called per frame by the rendering pipeline.
2. Iterates through sorted polygons (from farthest to nearest).
3. For each polygon, walks its object list (stored in game world).
4. Calls `build_render_object()` for each object: projects to 2D, handles 3D models.
5. Calls `build_base_node_list()` to find visibility nodes the object spans.
6. Calls `sort_render_object_into_tree()` to depth-sort within those nodes.
7. Calls `build_aggregate_render_object_clipping_window()` to merge clipping regions.
8. Result: `RenderObjects` vector populated with depth-sorted, clipped objects ready for rendering.

**3D model support:** If OpenGL is active and a 3D model is present, `FindProjectedBoundingBox()` computes a synthetic sprite bounding box and lighting info.

## External Dependencies

- **Notable includes:**
  - `cseries.h` ΓÇö utility macros (PIN, TEST_FLAG, WRAP_HIGH/WRAP_LOW, assert, etc.)
  - `map.h` ΓÇö polygon, object, endpoint, light source data structures and accessors
  - `lightsource.h` ΓÇö `get_light_intensity()`
  - `media.h` ΓÇö `get_media_data()`, media height lookups
  - `RenderPlaceObjs.h` ΓÇö class definition
  - `OGL_Setup.h` ΓÇö `OGL_GetModelData()`, 3D model structures
  - `ChaseCam.h` ΓÇö `GetChaseCamData()`
  - `player.h` ΓÇö `current_player`

- **Key external symbols (defined elsewhere):**
  - `get_object_data(index)` ΓÇö returns object from global object list
  - `get_polygon_data(index)` ΓÇö returns polygon from global map
  - `get_endpoint_data(index)` ΓÇö returns endpoint from global map
  - `get_light_intensity(index)` ΓÇö returns light intensity value
  - `extended_get_shape_information(collection, shape)` ΓÇö sprite/shape metadata
  - `OGL_GetModelData(collection, shape, &ModelSequence)` ΓÇö 3D model lookup
  - `extended_get_shape_bitmap_and_shading_table(...)` ΓÇö texture and shading lookups
  - `instantiate_rectangle_transfer_mode(...)` ΓÇö transfer mode animation setup
  - `current_player` ΓÇö global player reference
  - `cosine_table[]`, `sine_table[]` ΓÇö pre-computed trig tables (TRIG_MAGNITUDE scale)
  - `isqrt(x)` ΓÇö fast integer square root
