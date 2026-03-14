# Source_Files/RenderOther/OverheadMapRenderer.cpp

## File Purpose
Implements the overhead map renderer for the Aleph One game engine, displaying a top-down view of the game world. Renders polygons, lines, entities, annotations, and paths on the map; handles special checkpoint map mode by generating a temporary view of unexplored areas.

## Core Responsibilities
- Render polygons with color coding based on type (platforms, water, lava, sewage, goo, hills, damage zones)
- Draw map lines with elevation-change and solid-wall coloring
- Transform world coordinates to screen space and determine visibility
- Render game entities (players, monsters, items, projectiles) with team/type colors
- Display map annotations and level names
- Visualize pathfinding paths for debugging
- Generate and manage "false automap" state for checkpoint map rendering (revealing only accessible areas)
- Apply configuration-based display toggles (show aliens/items/projectiles/paths)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `polygon_data` | struct | Polygon definition with vertex indices, type, media, floor/ceiling heights |
| `line_data` | struct | Line segment with endpoint indices, polygon owners, elevation data |
| `media_data` | struct | Media type (water, lava, etc.) with height and properties |
| `endpoint_data` | struct | Map vertex with world position and transformed screen position |
| `map_annotation` | struct | Text label with location, polygon index, and annotation type |
| `object_data` | struct | Game entity (monster, item, projectile) with location and properties |
| `overhead_map_data` | struct | Render control: origin, scale, screen dimensions (from header) |
| `OvhdMap_CfgDataStruct` | struct | Configuration: colors, fonts, entity shapes, visibility flags (from header) |
| `world_point2d` | typedef | 2D coordinate (x, y) in world space or screen space |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `dynamic_world` | pointer | global | Current frame's world state; accessed for polygon/line/object counts and tick counter |
| `static_world` | pointer | global | Static world data; accessed for level name |
| `objects` | array | global | Game object instances; iterated to render monsters, items, projectiles |
| `saved_objects` | array | global | Checkpoint map state; iterated for goal location rendering |
| `automap_lines` | byte buffer | global | Visibility bitmask for lines in normal automap |
| `automap_polygons` | byte buffer | global | Visibility bitmask for polygons in normal automap |
| `saved_automap_lines` | byte pointer | member (class) | Backup of automap lines during checkpoint map generation |
| `saved_automap_polygons` | byte pointer | member (class) | Backup of automap polygons during checkpoint map generation |
| `local_player` | pointer | global | Local player data; checked for team visibility in omniscient mode |
| `ConfigPtr` | pointer | member (class) | Configuration data pointer; accessed for colors, fonts, and display settings |

## Key Functions / Methods

### Render
- **Signature:** `void OverheadMapClass::Render(overhead_map_data& Control)`
- **Purpose:** Main rendering routine; orchestrates all overhead map drawing in one frame.
- **Inputs:** `Control` structure with origin, scale, half-width/height, rendering mode, and polygon index.
- **Outputs/Return:** None; side effects include drawing to backend (via virtual methods).
- **Side effects:** Calls `begin_overall()`, `begin/end_polygons()`, `begin/end_lines()`, calls `transform_endpoints_for_overhead_map()`, reads and modifies render flags on endpoints/polygons/lines via `TEST_STATE_FLAG()` and `SET_STATE_FLAG()`, may call `generate_false_automap()` and `replace_real_automap()`, reads global `dynamic_world`, `objects`, `saved_objects`, `local_player`, modifies game options flags.
- **Calls (direct visible in file):** `transform_endpoints_for_overhead_map()`, `get_polygon_data()`, `get_line_data()`, `get_media_data()`, `get_platform_data()`, `draw_polygon()`, `draw_line()`, `get_next_map_annotation()`, `draw_annotation()`, `path_peek()`, `GetNumberOfPaths()`, `draw_path()`, `finish_path()`, `get_monster_data()`, `monster_index_to_player_index()`, `get_player_data()`, `draw_player()`, `draw_thing()`, `flood_map()`, `draw_map_name()`, `generate_false_automap()`, `replace_real_automap()`, `begin_overall()`, `end_overall()`, `begin_polygons()`, `end_polygons()`, `begin_lines()`, `end_lines()`.
- **Notes:** Respects game options flags (show aliens, items, projectiles). Skips entity rendering in checkpoint map mode. Uses macro `WORLD_TO_SCREEN()` for coordinate transformation. Conditionally applies `ConfigPtr->ShowPaths`, `ShowAliens`, `ShowItems`, `ShowProjectiles` based on configuration.

### transform_endpoints_for_overhead_map
- **Signature:** `void OverheadMapClass::transform_endpoints_for_overhead_map(overhead_map_data& Control)`
- **Purpose:** Transform all world endpoints to screen space and mark which polygons are on-screen.
- **Inputs:** `Control` with origin, scale, and screen dimensions.
- **Outputs/Return:** None; modifies render state flags.
- **Side effects:** Writes to `endpoint->transformed` (x, y); sets `_endpoint_on_automap` flag on endpoints and `_polygon_on_automap` flag on polygons. Reads endpoint positions and polygon vertex indices.
- **Calls (direct visible in file):** `get_endpoint_data()`, `SET_STATE_FLAG()`, `TEST_STATE_FLAG()`.
- **Notes:** Uses `WORLD_TO_SCREEN()` macro. Two-pass: first transforms all endpoints and marks visibility; second marks polygons visible if any vertex is on-screen.

### generate_false_automap
- **Signature:** `void OverheadMapClass::generate_false_automap(short polygon_index)`
- **Purpose:** Create a temporary automap that shows only areas reachable from a given polygon (for checkpoint maps).
- **Inputs:** `polygon_index` ΓÇô starting polygon for flood fill.
- **Outputs/Return:** None; allocates and modifies global automap buffers.
- **Side effects:** Allocates `saved_automap_lines` and `saved_automap_polygons` as backups; zeroes global `automap_lines` and `automap_polygons` buffers; calls `flood_map()` repeatedly with `false_automap_cost_proc()` callback to mark reachable areas.
- **Calls (direct visible in file):** `new byte[]`, `memcpy()`, `memset()`, `flood_map()`, `delete[]`.
- **Notes:** Calculates buffer size in bytes using `(count/8 + ((count%8)?1:0))*sizeof(byte)` for bit packing.

### replace_real_automap
- **Signature:** `void OverheadMapClass::replace_real_automap(void)`
- **Purpose:** Restore the original automap state after checkpoint map rendering.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Copies `saved_automap_lines` and `saved_automap_polygons` back to global `automap_lines` and `automap_polygons`; deallocates saved buffers and nullifies pointers.
- **Calls (direct visible in file):** `memcpy()`, `delete[]`.
- **Notes:** Conditional cleanup; only restores if buffers were allocated.

### false_automap_cost_proc
- **Signature:** `int32 OverheadMapClass::false_automap_cost_proc(short source_polygon_index, short line_index, short destination_polygon_index, void *caller_data)`
- **Purpose:** Cost function for flood-fill in checkpoint map; prevents passage through secret platforms and marks polygons/lines in the automap.
- **Inputs:** Source and destination polygon indices, line index, caller data (unused).
- **Outputs/Return:** Cost (1 for normal traversal, -1 to block passage).
- **Side effects:** Calls `ADD_LINE_TO_AUTOMAP()` and `ADD_POLYGON_TO_AUTOMAP()` macros to set bits in automap buffers. Reads platform data to check secret/door flags.
- **Calls (direct visible in file):** `get_polygon_data()`, `get_platform_data()`, `ADD_LINE_TO_AUTOMAP()`, `ADD_POLYGON_TO_AUTOMAP()`.
- **Notes:** Static method so it can be passed as a callback. Blocks passage from and to secret platforms (even if they are doors). Used as callback for `flood_map()` with breadth-first mode.

## Control Flow Notes
`Render()` is the main entry point, called during the game's render phase (likely from `render_overhead_map()` in render.h). It orchestrates coordinate transformation, polygon/line/entity rendering, and special checkpoint map handling. The checkpoint mode (`_rendering_checkpoint_map`) triggers `generate_false_automap()` before rendering and `replace_real_automap()` after, to show only accessible areas. Normal mode (`_rendering_game_map`) includes the level name overlay.

## External Dependencies
- **Includes:** `cseries.h` (core types), `OverheadMapRenderer.h` (class and config struct), `flood_map.h` (flood_map function), `media.h` (media_data struct), `platforms.h` (platform_data, macros), `player.h` (player_data, monster_index_to_player_index), `render.h` (render flags, view_data), `string.h`, `stdlib.h`, `limits.h`.
- **External Symbols:**
  - `dynamic_world` (current world state)
  - `static_world` (static data)
  - `objects` (game entity array)
  - `saved_objects` (checkpoint objects)
  - `automap_lines`, `automap_polygons` (visibility state)
  - `local_player` (player data)
  - `get_polygon_data()`, `get_line_data()`, `get_media_data()`, `get_endpoint_data()`, `get_platform_data()`, `get_player_data()`, `get_monster_data()` (accessors)
  - `get_next_map_annotation()`, `path_peek()`, `GetNumberOfPaths()` (iteration/lookup)
  - `monster_index_to_player_index()` (conversion)
  - `flood_map()` (pathfinding/flood fill)
  - Macros: `POLYGON_IS_IN_AUTOMAP()`, `TEST_STATE_FLAG()`, `SET_STATE_FLAG()`, `LINE_IS_IN_AUTOMAP()`, `LINE_IS_SOLID()`, `LINE_IS_VARIABLE_ELEVATION()`, `LINE_IS_LANDSCAPED()`, `PLATFORM_IS_SECRET()`, `PLATFORM_IS_DOOR()`, `POLYGON_IS_DETACHED()`, `OBJECT_IS_INVISIBLE()`, `GET_OBJECT_OWNER()`, `MONSTER_IS_PLAYER()`, `SLOT_IS_USED()`, `GET_GAME_OPTIONS()`, `WORLD_TO_SCREEN()`, `WORLD_ONE`, `NONE`, `UNONE`, `ADD_LINE_TO_AUTOMAP()`, `ADD_POLYGON_TO_AUTOMAP()`.
