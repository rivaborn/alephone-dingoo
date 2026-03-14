# Source_Files/GameWorld/flood_map.h

## File Purpose
Header file declaring the flood-map pathfinding system for the Aleph One game engine. Provides interfaces for computing navigable paths between map polygons using multiple flood-fill traversal strategies and custom cost functions.

## Core Responsibilities
- Define flood-fill traversal modes (depth-first, breadth-first, flagged breadth-first, best-first)
- Declare pathfinding memory allocation and initialization
- Provide path creation/deletion/traversal operations
- Declare flood-map computation, reversal, and depth queries
- Support random node selection from flood results

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `cost_proc_ptr` | typedef (function pointer) | Custom cost computation callback: `int32 (*)(short source_polygon, short line, short dest_polygon, void *caller_data)` |

## Global / File-Static State
None.

## Key Functions / Methods

### allocate_pathfinding_memory
- **Purpose:** Initialize pathfinding subsystem memory
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Allocates global memory pools for path data structures
- **Calls:** Not visible in this file

### new_path
- **Purpose:** Compute a path from source to destination polygon
- **Inputs:** source point, source polygon index, destination point, destination polygon index, minimum separation distance, optional cost procedure, caller data
- **Outputs/Return:** short (path index, or error code)
- **Side effects:** Allocates/stores path in global path list
- **Notes:** Cost procedure defaults to destination polygon area if NULL

### move_along_path
- **Purpose:** Advance along a computed path
- **Inputs:** path index, pointer to point to update
- **Outputs/Return:** bool (success)
- **Side effects:** Updates caller's point coordinate

### delete_path
- **Purpose:** Deallocate a path
- **Inputs:** path index
- **Outputs/Return:** void
- **Side effects:** Frees path resources

### allocate_flood_map_memory
- **Purpose:** Initialize flood-map memory structures
- **Inputs:** None
- **Outputs/Return:** void

### flood_map
- **Purpose:** Compute reachable polygons from a starting point with cost limit
- **Inputs:** starting polygon index, maximum traversal cost, optional cost procedure, flood mode, caller data (interpreted per mode)
- **Outputs/Return:** short (flood result count or status)
- **Notes:** Flagged mode interprets caller_data as int32* to 4-byte flag array; NULL cost_proc uses polygon area

### reverse_flood_map / flood_depth / choose_random_flood_node
- **Purpose:** Query/manipulate flood-map results
- **Notes:** Return types and exact semantics not fully inferable from signature alone

## Control Flow Notes
This file supports the init/frame loop: memory allocation at startup, `new_path`/`flood_map` during navigation queries, path traversal during movement updates, cleanup on shutdown.

## External Dependencies
- **Type imports (defined elsewhere):** `int32`, `short`, `bool`, `world_point2d`, `world_distance`, `world_vector2d`
- **Implicit dependencies:** Polygon/line index system (world topology model)
