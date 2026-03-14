# Source_Files/GameWorld/pathfinding.cpp

## File Purpose

Implements pathfinding for non-player characters (monsters) in the game world. Maintains a dynamic pool of pre-allocated paths that entities traverse sequentially. Uses a flood-fill algorithm to find routes through the polygon adjacency graph.

## Core Responsibilities

- Allocate and manage a pool of path objects with configurable maximum count
- Create new paths via flood-fill search from source to destination polygon
- Support both deterministic (goal-directed) and random path generation
- Advance entities one waypoint at a time along their assigned path
- Calculate safe passage points (midpoints of polygon-boundary lines)
- Validate path integrity in DEBUG builds

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `path_definition` | struct | Stores a single path: current step index, total step count, and array of up to 63 waypoint locations |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `paths` | `path_definition*` | static | Dynamic array of path objects; allocated once per engine initialization |
| `path_validation_area` | `byte*` | static | (DEBUG only) Buffer for comparing path consistency across simulation runs |
| `path_validation_area_index` | `int32` | static | (DEBUG only) Write position in validation area |
| `path_run_count` | `short` | static | (DEBUG only) Counter to detect first validation pass vs. later consistency checks |

## Key Functions / Methods

### allocate_pathfinding_memory
- **Signature:** `void allocate_pathfinding_memory(void)`
- **Purpose:** Allocate (or reallocate) the paths array and validation buffer. Called during initialization and whenever `MAXIMUM_PATHS` changes via dynamic limits.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deletes and reallocates static arrays; asserts on allocation failure
- **Calls:** `get_dynamic_limit()` (for MAXIMUM_PATHS macro)
- **Notes:** Reentrant by design; safely handles multiple calls

### reset_paths
- **Signature:** `void reset_paths(void)`
- **Purpose:** Mark all paths as unused at the start of each world update cycle.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `step_count = NONE` for all paths; increments `path_run_count` in DEBUG mode
- **Calls:** None
- **Notes:** Called before pathfinding requests each frame

### new_path
- **Signature:** `short new_path(world_point2d *source_point, short source_polygon_index, world_point2d *destination_point, short destination_polygon_index, world_distance minimum_separation, cost_proc_ptr cost, void *data)`
- **Purpose:** Create a new path from source to destination (or random direction if destination is NONE). Returns path index or NONE if no slot available.
- **Inputs:** Source and destination locations/polygons; minimum clearance from walls; cost function and custom caller data for flood-fill
- **Outputs/Return:** `short` path index (0 to MAXIMUM_PATHSΓêÆ1) or NONE
- **Side effects:** Allocates a path slot; calls flood-fill to search polygon graph; populates path waypoints
- **Calls:** `flood_map()`, `reverse_flood_map()`, `flood_depth()`, `choose_random_flood_node()`, `calculate_midpoint_of_shared_line()`
- **Notes:** Always returns a path if a free slot exists, even if destination is unreachable (returns clipped random path). Fills path waypoints in reverse order. Clips path to `MAXIMUM_POINTS_PER_PATH` if too long.

### move_along_path
- **Signature:** `bool move_along_path(short path_index, world_point2d *p)`
- **Purpose:** Retrieve the next waypoint on a path and advance the path cursor.
- **Inputs:** Path index; output pointer for waypoint
- **Outputs/Return:** `bool` true if this was the final waypoint (path exhausted), false otherwise
- **Side effects:** Increments `current_step`; marks path as unused (NONE) when exhausted
- **Calls:** None
- **Notes:** Entity must call repeatedly until true is returned

### delete_path
- **Signature:** `void delete_path(short path_index)`
- **Purpose:** Explicitly free a path slot (mark as unused).
- **Inputs:** Path index
- **Outputs/Return:** None
- **Side effects:** Sets path `step_count = NONE`; asserts on invalid index or already-deleted path
- **Calls:** None
- **Notes:** Optional; paths auto-free when exhausted via `move_along_path()`

### calculate_midpoint_of_shared_line (private)
- **Signature:** `static void calculate_midpoint_of_shared_line(short polygon1, short polygon2, world_distance minimum_separation, world_point2d *midpoint)`
- **Purpose:** Compute a safe passage point on the line shared by two adjacent polygons, respecting elevation endpoints.
- **Inputs:** Two adjacent polygon indices; minimum clearance distance; output pointer
- **Outputs/Return:** None (result in `*midpoint`)
- **Side effects:** None
- **Calls:** `find_shared_line()`, `get_line_data()`, `get_endpoint_data()`, `global_random()`
- **Notes:** If line too small for clearance, returns true line midpoint; else returns offset point using random bias to vary paths

### path_peek (debug)
- **Signature:** `world_point2d *path_peek(short path_index, short *step_count)`
- **Purpose:** Inspect a path's waypoints and step count (used by debug overlay).
- **Inputs:** Path index; output pointer for step count
- **Outputs/Return:** Pointer to path's waypoint array (or NULL if path unused)
- **Side effects:** None
- **Calls:** None

### GetNumberOfPaths
- **Signature:** `short GetNumberOfPaths()`
- **Purpose:** Return the current maximum path count (from dynamic limits).
- **Inputs:** None
- **Outputs/Return:** `short` MAXIMUM_PATHS
- **Side effects:** None
- **Calls:** (via MAXIMUM_PATHS macro) `get_dynamic_limit()`

## Control Flow Notes

**Initialization:** `allocate_pathfinding_memory()` called during engine startup; reallocated if dynamic limits change.

**Per-frame:** `reset_paths()` called before updating monster AI; each monster calls `new_path()` to request a route, then calls `move_along_path()` each tick until exhausted.

**Pathfinding algorithm:** Flood-fill breadth-first from source polygon, expanding outward until destination is found (or random node chosen). `reverse_flood_map()` then backtracks to reconstruct the route.

## External Dependencies

- **`flood_map.h`:** `flood_map()`, `reverse_flood_map()`, `flood_depth()`, `choose_random_flood_node()` ΓÇö core graph search; `cost_proc_ptr` callback type
- **`map.h`:** `find_shared_line()`, `get_line_data()`, `get_endpoint_data()`, polygon/line/endpoint data accessors
- **`dynamic_limits.h`:** `get_dynamic_limit()` for runtime limit configuration
- **`cseries.h`:** Macros `obj_set()`, `objlist_copy()`, assertion/debug utilities
- **Built-in C headers:** `<string.h>`, `<stdlib.h>`, `<limits.h>`
