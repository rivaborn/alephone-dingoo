# Source_Files/GameWorld/flood_map.cpp

## File Purpose
Implements a stateful graph search algorithm for pathfinding across polygon-based game world geometry. Supports multiple search strategies (breadth-first, best-first, depth-first) with customizable cost evaluation and node selection.

## Core Responsibilities
- Manage dynamic node allocation and search tree construction for polygon traversal
- Execute multiple flood-fill search algorithms (best-first, breadth-first, flagged breadth-first)
- Track visited polygons to prevent revisits and enable path reconstruction
- Support reverse path traversal (backtracking from destination to source)
- Provide depth calculation and biased random node selection for pathfinding algorithms

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `node_data` | struct | 16-byte node in search tree; tracks parent, polygon, cost, depth, expansion state, and user flags |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `node_count` | short | static | Count of nodes currently in the search tree |
| `last_node_index_expanded` | short | static | Index of the most recently expanded node; used for path reconstruction and depth queries |
| `nodes` | `struct node_data*` | static | Dynamically allocated array of nodes; max 255 entries |
| `visited_polygons` | `short*` | static | Array mapping polygon indices to node indices; tracks exploration state |

## Key Functions / Methods

### allocate_flood_map_memory
- **Signature:** `void allocate_flood_map_memory(void)`
- **Purpose:** Allocate/reallocate dynamic memory for flood search state. Made reentrant to support map reloading.
- **Inputs:** None (uses global constants: `MAXIMUM_FLOOD_NODES`, `MAXIMUM_POLYGONS_PER_MAP`)
- **Outputs/Return:** None
- **Side effects:** Deletes and reallocates `nodes` and `visited_polygons` arrays; asserts on allocation failure
- **Calls:** `delete[]`, `new`
- **Notes:** Reentrant by design; safely handles repeated calls on map loads

### flood_map
- **Signature:** `short flood_map(short first_polygon_index, int32 maximum_cost, cost_proc_ptr cost_proc, short flood_mode, void *caller_data)`
- **Purpose:** Execute one step of graph search; returns the next polygon to visit or NONE when exhausted.
- **Inputs:**
  - `first_polygon_index`: Start polygon (NONE to continue prior search)
  - `maximum_cost`: Cost threshold for node expansion
  - `cost_proc`: Optional callback to compute edge costs (defaults to polygon area)
  - `flood_mode`: Search strategy (_best_first, _breadth_first, _flagged_breadth_first, _depth_first)
  - `caller_data`: User callback data or flags buffer
- **Outputs/Return:** Next unexpanded polygon index within cost budget, or NONE if none found
- **Side effects:** Modifies global search state (`node_count`, `last_node_index_expanded`, `nodes`, `visited_polygons`); calls `cost_proc` for each adjacent polygon
- **Calls:** `objlist_set`, `add_node`, `get_polygon_data`
- **Notes:**
  - Stateful: call with `first_polygon_index!=NONE` to initialize a new search; subsequent calls with `NONE` continue
  - Best-first guarantees optimal paths; breadth-first is faster for large maps
  - Returns NONE when no unexpanded node exists with cost < maximum_cost

### reverse_flood_map
- **Signature:** `short reverse_flood_map(void)`
- **Purpose:** Backtrack one step along the search tree from the last expanded node, reconstructing the path.
- **Inputs:** None (uses `last_node_index_expanded`)
- **Outputs/Return:** Polygon index of the parent node; NONE if already at start
- **Side effects:** Updates `last_node_index_expanded` to point to parent
- **Calls:** None
- **Notes:** Used for pathfinding; call repeatedly to walk from destination back to source

### flood_depth
- **Signature:** `short flood_depth(void)`
- **Purpose:** Return the depth (polygon count) from start to the last expanded node.
- **Inputs:** None (uses `last_node_index_expanded`)
- **Outputs/Return:** Depth as int16; 0 if no node expanded
- **Side effects:** None (read-only)
- **Calls:** None
- **Notes:** Useful for distance metrics in pathfinding

### choose_random_flood_node
- **Signature:** `void choose_random_flood_node(world_vector2d *bias)`
- **Purpose:** Randomly select an expanded node, optionally biased toward a direction.
- **Inputs:** `bias` pointer to direction vector (NULL for unbiased selection)
- **Outputs/Return:** None (modifies `last_node_index_expanded`)
- **Side effects:** Updates `last_node_index_expanded` to a random expanded node; uses `global_random()` and polygon geometry lookups
- **Calls:** `global_random`, `find_center_of_polygon`
- **Notes:** Uses dot product to check directional bias; retries up to 10 times if biased; asserts node_count >= 1

### add_node (private)
- **Signature:** `static void add_node(short parent_node_index, short polygon_index, short depth, int32 cost, int32 user_flags)`
- **Purpose:** Conditionally add or update a polygon in the search tree.
- **Inputs:**
  - `parent_node_index`: Index of the node from which we reached this polygon
  - `polygon_index`: Destination polygon
  - `depth`: Depth in the tree
  - `cost`: Cumulative cost to reach this node
  - `user_flags`: Custom per-node flags (for _flagged_breadth_first mode)
- **Outputs/Return:** None
- **Side effects:** Inserts or updates entry in `nodes` and `visited_polygons` arrays; silently drops if node limit reached
- **Calls:** `get_polygon_data`
- **Notes:**
  - Avoids duplicate polygon entries: if already in the list with lower cost, replaces it (better-first invariant)
  - Skips nodes already expanded (backtracking guard)
  - Only adds if node_count < MAXIMUM_FLOOD_NODES

## Control Flow Notes
This is a **stateful incremental search** designed for repeated calls over a single frame or game loop:

1. **Initialization:** Call `allocate_flood_map_memory()` once per map load
2. **Search cycle:**
   - Call `flood_map(start_polygon, cost_limit, cost_fn, mode, data)` to initialize and return first unexpanded node
   - Repeatedly call `flood_map(NONE, ...)` to step through search; each call expands one node and returns the next candidate
   - Use `reverse_flood_map()` repeatedly to reconstruct the path from destination back to source
   - Use `flood_depth()` to query path length
3. **Optional:** `choose_random_flood_node()` for AI pathfinding with noise/randomness

The search maintains expanded/unexpanded node state via bit flags (`NODE_IS_EXPANDED` macro); adjacent polygons are discovered via `polygon_data.adjacent_polygon_indexes[]`.

## External Dependencies
- **Includes:**
  - `cseries.h`: Utility macros and types
  - `map.h`: World geometry (polygon_data, endpoints, lines, etc.)
  - Standard C: `string.h`, `stdlib.h`, `limits.h`
- **Functions defined elsewhere:**
  - `get_polygon_data(int16)`: Accessor from map.h
  - `find_center_of_polygon(int16, world_point2d*)`: Utility from map.h
  - `global_random()`: RNG (defined elsewhere)
  - `objlist_set()`: Array initialization utility
- **Global data:**
  - `dynamic_world`: World state containing polygon_count and other runtime data
