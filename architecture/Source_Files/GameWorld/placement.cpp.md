# Source_Files/GameWorld/placement.cpp

## File Purpose

Manages object (monster and item) placement and spawning in the game world. Handles initial level population, periodic respawning based on frequency rules, player spawn point selection, and validation of spawn locations.

## Core Responsibilities

- Load serialized placement frequency data for monsters and items into memory
- Place initial monsters and items according to map-defined placement rules
- Periodically recreate monsters and items when populations fall below minimum thresholds
- Track live object counts and manage minimum/maximum spawn constraints
- Find valid spawn locations from predefined map points and random locations
- Select optimal player starting positions considering proximity to monsters and other players
- Mark and preload monster collections and sounds required by the current map

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `object_frequency_definition` | struct | Defines spawn parameters: initial count, min/max, random chance, reappearance flags |
| `object_location` | struct | Position + polygon + facing angles for spawning objects |
| `map_object` | struct | Saved initial placements from the map file; source for predefined spawn locations |
| `polygon_data` | struct | World geometry; used to validate spawn polygons |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `object_placement_info` | `object_frequency_definition[2*MAXIMUM_OBJECT_TYPES]` | static | Unified array: first half is items, second half is monsters |
| `monster_placement_info` | `object_frequency_definition*` | static | Points into `object_placement_info+MAXIMUM_OBJECT_TYPES` |
| `item_placement_info` | `object_frequency_definition*` | static | Points into `object_placement_info` (first half) |

## Key Functions / Methods

### load_placement_data
- **Signature:** `void load_placement_data(uint8 *_monsters, uint8 *_items)`
- **Purpose:** Deserialize placement frequency data from map file and initialize global placement tables.
- **Inputs:** Byte streams containing packed monster and item frequency definitions.
- **Outputs/Return:** None; updates static `object_placement_info`, `monster_placement_info`, `item_placement_info`.
- **Side effects:** Clears all placement arrays, unpacks serialized data, zeroes Marine (monster #0) entry, validates counts in debug builds.
- **Calls:** `objlist_clear()`, `unpack_object_frequency_definition()`, `obj_clear()`.
- **Notes:** Excludes Marine from placement. Includes debug validation for negative counts and improper Marine config.

### place_initial_objects
- **Signature:** `void place_initial_objects(void)`
- **Purpose:** Spawn initial monsters and items at level start according to placement data.
- **Inputs:** None (reads from global `monster_placement_info`, `item_placement_info`).
- **Outputs/Return:** None; creates monsters and items via `add_objects()`.
- **Side effects:** Populates `dynamic_world->current_monster_count`, `current_item_count`; initializes random spawn counters.
- **Calls:** `add_objects()`, `GET_GAME_OPTIONS()`.
- **Notes:** Respects `_monsters_replenish` game option. Starts monster loop at index 1 (skips Marine). Initializes `random_monsters_left` and `random_items_left` for periodic spawning.

### recreate_objects
- **Signature:** `void recreate_objects(void)`
- **Purpose:** Periodic callback to respawn monsters and items if populations drop below minimums or random spawns are triggered.
- **Inputs:** None (reads from `dynamic_world->tick_count`).
- **Outputs/Return:** None.
- **Side effects:** Updates static delay; calls `_recreate_objects()` every `NUMBER_OF_TICKS_BETWEEN_RECREATION` (15 seconds).
- **Calls:** `_recreate_objects()`, `GET_GAME_OPTIONS()`.
- **Notes:** Resets delay if `tick_count` goes backward (new game started). Only checks monsters if `_monsters_replenish` is set.

### object_was_just_added
- **Signature:** `void object_was_just_added(short object_class, short object_type)`
- **Purpose:** Track that a new object was spawned; increment live count.
- **Inputs:** `object_class` (_object_is_monster or _object_is_item), `object_type` (monster/item subtype).
- **Outputs/Return:** None.
- **Side effects:** Increments `dynamic_world->current_monster_count[type]` or `current_item_count[type]`.
- **Calls:** None.
- **Notes:** Called from `new_monster()` and `new_item()`.

### object_was_just_destroyed
- **Signature:** `void object_was_just_destroyed(short object_class, short object_type)`
- **Purpose:** Track object destruction; decrement live count and trigger respawn if population falls below minimum.
- **Inputs:** Object class and type.
- **Outputs/Return:** None.
- **Side effects:** Decrements live count; compares to minimum and calls `add_objects()` if needed. Handles special case where destroyed item was never placed on map.
- **Calls:** `add_objects()`, `GET_GAME_OPTIONS()`.
- **Notes:** Only respawns items unconditionally; monsters only respawn if `_monsters_replenish` is active.

### get_random_player_starting_location_and_facing
- **Signature:** `short get_random_player_starting_location_and_facing(short max_player_index, short team, struct object_location *location)`
- **Purpose:** Select a player spawn point that is far from monsters and existing players.
- **Inputs:** Max player index (for visibility checks), team (or NONE), output location pointer.
- **Outputs/Return:** Starting location index; fills `*location` with position, polygon, facing.
- **Side effects:** None.
- **Calls:** `get_player_starting_location_and_facing()`, `point_is_player_visible()`, `point_is_monster_visible()`.
- **Notes:** Falls back to any team if no team-specific starts exist. Weighs player distance twice as heavily as monster distance. If all locations have monsters/players, returns last evaluated location.

---

## Control Flow Notes

**Initialization phase (game start):**
- `load_placement_data()` called by game WAD loader
- `place_initial_objects()` called during level entry; spawns all initial monsters/items
- `mark_all_monster_collections()` and `load_all_monster_sounds()` preload assets

**Per-frame update:**
- `recreate_objects()` called periodically (~every 15 seconds) to spawn replacements
- `object_was_just_added()` / `object_was_just_destroyed()` called by monster/item managers to track population

**Player spawn:**
- `get_random_player_starting_location_and_facing()` called when respawning a player

## External Dependencies

- **map.h**: `object_location`, `polygon_data`, `object_data`, `saved_objects`, `dynamic_world`, geometry functions
- **monsters.h**: `new_monster()`, `activate_monster()`, `find_closest_appropriate_target()`, monster type enums
- **items.h**: `new_item()`, item type enums
- **cseries.h**: `global_random()`, memory utilities
- **map.c/world.c** (defined elsewhere): `point_is_player_visible()`, `point_is_monster_visible()`, `find_center_of_polygon()`, `find_new_object_polygon()`, `get_polygon_data()`, `get_object_data()`, `GET_GAME_OPTIONS()`, `get_player_starting_location_and_facing()`
