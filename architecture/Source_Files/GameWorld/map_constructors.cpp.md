# Source_Files/GameWorld/map_constructors.cpp

## File Purpose

Core map geometry construction and precalculation system for the Aleph One engine (Marathon). Handles creation of polygons, sides, lines, and endpoints; calculates geometric properties (area, adjacencies, exclusion zones); and implements binary packing/unpacking for save file serialization.

## Core Responsibilities

- Create new map geometry primitives (sides, polygons, lines, endpoints)
- Calculate and cache geometric properties: polygon area, clockwise endpoint ordering, adjacent polygons/sides
- Classify side types (_full_side, _high_side, _low_side, _split_side) based on adjacent polygon heights
- Precalculate spatial indexes for collision detection (exclusion zones, neighbor lists)
- Assign lightsources to sides based on side type and polygon lighting
- Flood-fill algorithm to find intersecting endpoints/lines/polygons within a separation distance
- Serialize/deserialize all map data structures to/from binary streams (big-endian format)
- Manage dynamic map index buffer allocation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `intersecting_flood_data` | struct | Temporary context for flood-fill geometry queries; holds original polygon, search center, and separation threshold |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DoIncorrectCountVWarn` | const bool | file-static | Debug flag: enable/disable warnings for incomplete neighbor lists (set to true) |
| `map_index_buffer_count` | int32 | file-static | Tracks capacity of the map index buffer; set via `set_map_index_buffer_size()` |
| `LineIndices` | `vector<short>` | file-static | Temporary list of line indices found during flood-fill (reused across calls) |
| `EndpointIndices` | `vector<short>` | file-static | Temporary list of endpoint indices found during flood-fill (reused across calls) |
| `PolygonIndices` | `vector<short>` | file-static | Temporary list of polygon indices found during flood-fill (reused across calls) |

## Key Functions / Methods

### recalculate_side_type
- **Signature:** `void recalculate_side_type(short side_index)`
- **Purpose:** Determine and set the type of a side based on adjacent polygon heights; updates `side->type`.
- **Inputs:** side_index (identifier into SideList)
- **Outputs/Return:** None (modifies side in-place)
- **Side effects:** Modifies side data in SideList; queries polygon/platform data
- **Calls:** `get_side_data()`, `find_adjacent_polygon()`, `get_polygon_data()`, `get_platform_data()`
- **Notes:** Handles platform types separately (uses min/max heights instead of static heights). Side type encodes wall configuration: NONE neighbor ΓåÆ _full_side; ceiling gap < floor gap ΓåÆ _split_side; ceiling gap only ΓåÆ _high_side; floor gap only ΓåÆ _low_side.

### new_side
- **Signature:** `short new_side(short polygon_index, short line_index)`
- **Purpose:** Create a new side linking a polygon to a line; append to SideList and return index.
- **Inputs:** polygon_index, line_index
- **Outputs/Return:** new side index
- **Side effects:** Appends to SideList; updates line's clockwise/counterclockwise side index; increments dynamic_world->side_count
- **Calls:** `get_line_data()`, `get_polygon_data()`, `SideList.push_back()`, `recalculate_redundant_side_data()`, `calculate_adjacent_sides()`, `recalculate_side_type()`
- **Notes:** Asserts that the side does not already exist on the line. Initializes textures to UNONE.

### recalculate_redundant_polygon_data
- **Signature:** `void recalculate_redundant_polygon_data(short polygon_index)`
- **Purpose:** Recompute derived polygon properties: clockwise endpoints, adjacent polygons, area, center, and side indices.
- **Inputs:** polygon_index
- **Outputs/Return:** None (modifies polygon in-place)
- **Side effects:** Modifies polygon's endpoint_indexes, adjacent_polygon_indexes, area, center, side_indexes
- **Calls:** `get_polygon_data()`, `POLYGON_IS_DETACHED()`, `calculate_clockwise_endpoints()`, `calculate_adjacent_polygons()`, `calculate_polygon_area()`, `find_center_of_polygon()`, `calculate_adjacent_sides()`
- **Notes:** Skipped for detached polygons. Contains TODO comments for uninitialized sound fields.

### recalculate_redundant_endpoint_data
- **Signature:** `void recalculate_redundant_endpoint_data(short endpoint_index)`
- **Purpose:** Calculate endpoint solidity, transparency, elevation, and adjacent floor/ceiling heights.
- **Inputs:** endpoint_index
- **Outputs/Return:** None (modifies endpoint in-place)
- **Side effects:** Updates endpoint flags and height fields
- **Calls:** `get_endpoint_data()`, `map_lines`, `LINE_IS_SOLID()`, `LINE_IS_TRANSPARENT()`, `LINE_IS_ELEVATION()`, `get_polygon_data()`, `SET_ENDPOINT_SOLIDITY()`, `SET_ENDPOINT_TRANSPARENCY()`, `SET_ENDPOINT_ELEVATION()`
- **Notes:** Iterates all lines to check if they include this endpoint; examines adjacent polygons for height extrema. Tracks which polygon provides the highest adjacent floor (supporting polygon).

### recalculate_redundant_line_data
- **Signature:** `void recalculate_redundant_line_data(short line_index)`
- **Purpose:** Compute line length, adjacent floor/ceiling heights, elevation flags, and landscaping status; recalculate both adjacent sides.
- **Inputs:** line_index
- **Outputs/Return:** None (modifies line and its sides in-place)
- **Side effects:** Updates line length, highest_adjacent_floor, lowest_adjacent_ceiling, and flags (elevation, variable_elevation, landscape, transparent)
- **Calls:** `get_line_data()`, `get_endpoint_data()`, `distance2d()`, `get_polygon_data()`, `recalculate_redundant_side_data()`, `get_side_data()`, `SET_LINE_*()` macros
- **Notes:** Handles edge cases where one or both adjacent polygons are NONE. Sets variable_elevation only if neither side is solid and at least one adjacent polygon is a platform.

### recalculate_redundant_side_data
- **Signature:** `void recalculate_redundant_side_data(short side_index, short line_index)`
- **Purpose:** Set up exclusion zone (inset quad), polygon reference, and line reference for a side.
- **Inputs:** side_index, line_index
- **Outputs/Return:** None (modifies side in-place)
- **Side effects:** Updates side.exclusion_zone, side.polygon_index, side.line_index
- **Calls:** `get_side_data()`, `get_line_data()`, `get_endpoint_data()`, `push_out_line()`
- **Notes:** Exclusion zone is a quad inset from the line's endpoints by MINIMUM_SEPARATION_FROM_WALL; used for collision detection.

### calculate_endpoint_polygon_owners
- **Signature:** `void calculate_endpoint_polygon_owners(short endpoint_index, short *first_index, short *index_count)`
- **Purpose:** Find all polygons that reference an endpoint; append indices to map_indexes array.
- **Inputs:** endpoint_index
- **Outputs/Return:** first_index (starting position in map_indexes), index_count (count appended)
- **Side effects:** Appends to map_indexes via `add_map_index()`; updates dynamic_world->map_index_count
- **Calls:** `map_polygons`, `add_map_index()`
- **Notes:** Not inferable from this file where results are used.

### guess_side_lightsource_indexes
- **Signature:** `void guess_side_lightsource_indexes(short side_index)`
- **Purpose:** Assign primary, secondary, and transparent lightsource indices to a side based on its type.
- **Inputs:** side_index
- **Outputs/Return:** None (modifies side in-place)
- **Side effects:** Updates side.primary_lightsource_index, side.secondary_lightsource_index, side.transparent_lightsource_index
- **Calls:** `get_side_data()`, `get_line_data()`, `get_polygon_data()`
- **Notes:** _full_side uses ceiling light; _high_side uses ceiling; _low_side uses floor; _split_side uses floor or ceiling depending on gap height vs. CONTINUOUS_SPLIT_SIDE_HEIGHT. Transparent always uses ceiling.

### precalculate_map_indexes
- **Signature:** `void precalculate_map_indexes(void)`
- **Purpose:** Main entry point: precalculate all spatial indexes (exclusion zones and neighbor lists) for all non-detached polygons.
- **Inputs:** None
- **Outputs/Return:** None (populates map_indexes array and polygon metadata)
- **Side effects:** Calls `find_intersecting_endpoints_and_lines()` twice per polygon (once for walls, once for projectiles); populates `LineIndices`, `EndpointIndices`, `PolygonIndices`; updates polygon.first_exclusion_zone_index, polygon.line_exclusion_zone_count, polygon.point_exclusion_zone_count, polygon.first_neighbor_index, polygon.neighbor_count; calls `precalculate_polygon_sound_sources()`
- **Calls:** `map_polygons`, `POLYGON_IS_DETACHED()`, `find_intersecting_endpoints_and_lines()`, `add_map_index()`, `precalculate_polygon_sound_sources()`
- **Notes:** Uses two separation distances: MINIMUM_SEPARATION_FROM_WALL and MINIMUM_SEPARATION_FROM_PROJECTILE. Calls `precalculate_polygon_sound_sources()` at the end (not defined in this file).

### find_intersecting_endpoints_and_lines
- **Signature:** `static void find_intersecting_endpoints_and_lines(short polygon_index, world_distance minimum_separation)`
- **Purpose:** Flood-fill from polygon to find nearby endpoints, lines, and polygons within minimum_separation; populate global vectors.
- **Inputs:** polygon_index, minimum_separation (distance threshold, squared internally)
- **Outputs/Return:** None (populates LineIndices, EndpointIndices, PolygonIndices vectors)
- **Side effects:** Clears and refills the three global index vectors; calls `flood_map()` with `intersecting_flood_proc` callback
- **Calls:** `intersecting_flood_data` (struct init), `flood_map()`, `find_center_of_polygon()`
- **Notes:** Uses breadth-first flood-fill with INT32_MAX cost (unlimited depth). Callback is defined but marked NEW_AND_BROKEN in code (not executed).

## Control Flow Notes

**Initialization phase (map load):**
- `precalculate_map_indexes()` is called once after map geometry is loaded to build all spatial structures.

**Per-element updates (runtime or editor):**
- Individual `recalculate_redundant_*()` functions can be called when a single element (polygon, endpoint, line, side) changes to update derived data.
- `recalculate_side_type()` is called when polygon heights or adjacencies change.

**Serialization (save/load):**
- `pack_*()` and `unpack_*()` functions serialize/deserialize individual data structures to/from byte streams in big-endian format.
- Used in save file I/O and network transmission.

Not inferable: when individual element recalculation is triggered during gameplay vs. editor use.

## External Dependencies

- **map.h** ΓÇô map data structure definitions (polygon_data, line_data, side_data, endpoint_data, object_data, dynamic_world, static_world, SideList, PolygonList, etc.)
- **flood_map.h** ΓÇô `flood_map()` function for spatial traversal
- **platforms.h** ΓÇô `platform_data`, `get_platform_data()` for platform-specific height queries
- **cseries.h** ΓÇô macros and utilities (obj_clear, TEST_FLAG, SET_FLAG, etc.)
- **Packing.h** ΓÇô `StreamToValue()`, `ValueToStream()`, `StreamToList()`, `ListToStream()`, `StreamToBytes()`, `BytesToStream()` macros for binary serialization
- **limits.h** ΓÇô INT16_MIN, INT16_MAX, INT32_MAX
- **vector** (STL) ΓÇô `vector<short>` containers

Notable symbols used but not defined here: `find_center_of_polygon()`, `distance2d()`, `push_out_line()`, `clockwise_endpoint_in_line()`, `find_adjacent_polygon()`, `flood_map()`, `get_platform_data()`, `precalculate_polygon_sound_sources()`, `add_map_index()`, various accessor macros.
