# Source_Files/GameWorld/platforms.cpp

## File Purpose

Implements the platform system for the Aleph One game engine, managing movable platforms (doors, elevators, platforms) in the game world. Handles platform physics, state transitions, player/monster interactions, geometry updates, and serialization.

## Core Responsibilities

- Create and initialize platforms from static configuration data
- Update all platforms each game tick (movement, obstruction handling, sound playback)
- Manage platform state activation/deactivation with cascading effects to adjacent platforms
- Detect obstruction when platforms move and handle reversal or blocking
- Adjust world geometry (polygon and endpoint heights) when platforms move
- Handle player and monster interactions (entry detection, state changes, key requirements)
- Evaluate platform accessibility for monster pathfinding
- Track media submersion state (floor/ceiling relative to liquids)
- Support XML-based runtime configuration of platform definitions
- Serialize/deserialize platform data for save games and network transmission

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `platform_data` | struct | Runtime state: position, flags, height ranges, geometry relationships |
| `static_platform_data` | struct | Static config: type, speed, delay, height bounds, flags |
| `platform_definition` | struct | Type definition: sounds, damage, key requirements (from platform_definitions.h) |
| `endpoint_owner_data` | struct | Adjacency metadata: adjacent polygon/line indices for each endpoint |
| `XML_PlatformParser` | class | XML element parser for platform definition overrides |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `platforms` | `platform_data*` (macro to vector) | global | Active platform instances indexed by platform_index |
| `PlatformList` | `vector<platform_data>` | global | Underlying dynamic array of all platforms |
| `original_platform_definitions` | `platform_definition*` | static | Backup copy of platform type definitions for XML reset |
| `PlatformParser` | `XML_PlatformParser` | static | Reusable XML parser instance |
| `PlatformsParser` | `XML_ElementParser` | static | Root XML element for platforms section |

## Key Functions / Methods

### new_platform
- Signature: `short new_platform(struct static_platform_data *data, short polygon_index)`
- Purpose: Create and register a new platform in the world
- Inputs: Static platform config, polygon index where platform resides
- Outputs/Return: Platform index (NONE if exceeds max), increments `platform_count`
- Side effects: Modifies polygon (sets permutation, type), calculates extrema, establishes endpoint ownership, calls `adjust_platform_endpoint_and_line_heights()`, `adjust_platform_for_media()`
- Calls: `get_polygon_data()`, `calculate_platform_extrema()`, `calculate_endpoint_polygon_owners()`, `calculate_endpoint_line_owners()`, `adjust_platform_endpoint_and_line_heights()`, `adjust_platform_for_media()`
- Notes: Asserts platform doesn't exceed limits; handles initially-active/extended flags; records endpoint relationships for later height recalculation

### update_platforms
- Signature: `void update_platforms(void)`
- Purpose: Main game loop entry point; advance all active platforms one tick
- Inputs: None (reads global `platforms`, `dynamic_world->platform_count`)
- Outputs/Return: None (modifies platform state, polygon heights, plays sounds)
- Side effects: Movement (height changes), obstruction detection, sound playback, adjacent platform cascading, light toggling, Lua callbacks, switch position updates
- Calls: `get_platform_data()`, `get_polygon_data()`, `get_platform_definition()`, `change_polygon_height()`, `adjust_platform_sides()`, `adjust_platform_endpoint_and_line_heights()`, `adjust_platform_for_media()`, `take_out_the_garbage()`, `set_adjacent_platform_states()`, `set_platform_state()`, `play_platform_sound()`, `L_Call_Platform_Activated()`, `assume_correct_switch_position()`
- Notes: Clamps height to min/max when fully extended/contracted; reverses or blocks on obstruction; deactivates at initial level or each level based on flags; complex state machine for movement phases

### set_platform_state
- Signature: `bool set_platform_state(short platform_index, bool state, short parent_platform_index)`
- Purpose: Change platform active/inactive, triggering cascading state changes
- Inputs: Platform index, desired state (true=active, false=inactive), parent platform (to avoid reactivation cycles)
- Outputs/Return: New platform state (true if now active)
- Side effects: Activates/deactivates lights, cascades to adjacent platforms, updates switch positions, calls Lua callbacks, sets activation flags
- Calls: `get_platform_data()`, `get_polygon_data()`, `set_light_status()`, `set_adjacent_platform_states()`, `L_Call_Platform_Activated()`, `assume_correct_switch_position()`, `play_platform_sound()`
- Notes: Per-tick state-change limit via `_platform_was_just_activated_or_deactivated` flag; respects `_platform_cannot_be_externally_deactivated` flag; complex cascading behavior for adjacent platforms

### adjust_platform_endpoint_and_line_heights
- Signature: `void adjust_platform_endpoint_and_line_heights(short platform_index)`
- Purpose: Recalculate endpoint/line heights after platform moved, updating geometry for adjacent polygons
- Inputs: Platform index
- Outputs/Return: None (modifies endpoint/line data in-place)
- Side effects: Updates line heights, transparency, solidity; endpoint heights, solidity, transparency
- Calls: `get_platform_data()`, `get_polygon_data()`, `get_endpoint_data()`, `get_line_data()`, `get_map_indexes()`
- Notes: Uses stored endpoint_owner relationships; handles lines with adjacent polygons and standalone lines; computes highest adjacent floor and lowest adjacent ceiling across multiple adjacent polygons

### play_platform_sound
- Signature: `static void play_platform_sound(short platform_index, short type)`
- Purpose: Play appropriate sound for platform event
- Inputs: Platform index, sound code (_obstructed_sound, _uncontrollable_sound, _starting_sound, _stopping_sound)
- Outputs/Return: None (triggers sound system)
- Side effects: Plays polygon sound, updates SoundManager ambient sources
- Calls: `get_platform_data()`, `get_platform_definition()`, `play_polygon_sound()`, `SoundManager::instance()->CauseAmbientSoundSourceUpdate()`
- Notes: Maps sound code to definition; uses platform direction (extending/contracting) for start/stop sounds

### player_touch_platform_state
- Signature: `void player_touch_platform_state(short player_index, short platform_index)`
- Purpose: Handle player action key press on platform (e.g., door interaction)
- Inputs: Player and platform indices
- Outputs/Return: None
- Side effects: Changes platform state, plays sound, consumes key item if required
- Calls: `get_platform_data()`, `get_platform_definition()`, `try_and_subtract_player_item()`, `set_platform_state()`, `play_platform_sound()`
- Notes: Checks player controllability and key requirements; respects deactivation locks; can reverse direction if obstructed or zero delay if stopped

### monster_can_enter_platform
- Signature: `short monster_can_enter_platform(short platform_index, short source_polygon_index, world_distance height, world_distance minimum_ledge_delta, world_distance maximum_ledge_delta)`
- Purpose: Evaluate whether monster can move from source to platform polygon
- Inputs: Platform, source polygon, monster height, ledge constraints
- Outputs/Return: Accessibility code (_platform_is_accessable, _platform_will_be_accessable, etc.)
- Side effects: None
- Calls: `get_polygon_data()`, `get_platform_data()`
- Notes: Considers current and min/max platform heights; special case for non-full-height platforms

### calculate_platform_extrema
- Signature: `static void calculate_platform_extrema(short platform_index, world_distance lowest_level, world_distance highest_level)`
- Purpose: Compute min/max floor and ceiling heights the platform can reach
- Inputs: Platform index, user-specified bounds (or NONE to auto-calculate)
- Outputs/Return: None (stores in platform structure)
- Side effects: None
- Calls: `get_platform_data()`, `get_polygon_data()`
- Notes: Considers adjacent polygon heights, platform direction (floor, ceiling, both), native polygon heights flag, extends-to-ceiling flag; handles split platforms (meet in center)

### Serialization functions
- `unpack_platform_data()`, `pack_platform_data()`, `unpack_static_platform_data()`, `pack_static_platform_data()`: Bidirectional conversion between binary streams and platform structures for saves/net

### XML support
- `XML_PlatformParser::Start()`, `HandleAttribute()`, `AttributesDone()`, `ResetValues()`: Parse and apply XML overrides to platform definitions
- `Platforms_GetParser()`: Return parser for integration into map XML loader

## Control Flow Notes

**Initialization** (`entering_map()` ΓåÆ map loading):
- `new_platform()` called for each platform in map
- Establishes geometry relationships, calculates extrema, registers with world

**Per-tick update** (`update_world()` ΓåÆ `update_platforms()`):
- Iterate active platforms; compute delta_height based on speed and direction
- Call `change_polygon_height()` to apply and check obstruction
- On success: update geometry via `adjust_platform_sides()` and `adjust_platform_endpoint_and_line_heights()`
- On obstruction: play sound, optionally reverse direction or block
- At extremes: deactivate if appropriate, cascade to adjacent platforms, play stop sound

**Interactions** (player action key / monster AI):
- `player_touch_platform_state()` or `platform_was_entered()` called
- May invoke `set_platform_state()` which cascades to adjacent platforms
- Lua hook `L_Call_Platform_Activated()` fired

**Cleanup** (level exit / save):
- Implicit via vector destruction

## External Dependencies

- **world.h**: `world_distance`, `world_point2d`, angle, trigonometry
- **map.h**: `polygon_data`, `endpoint_data`, `line_data`, `side_data`, `object_data`, world geometry accessors, change_polygon_height(), remove_map_object(), play_polygon_sound(), assume_correct_switch_position()
- **platforms.h**: Platform macros (FLAG tests/sets), `static_platform_data`, `platform_data`, `platform_definition`
- **lightsource.h**: `set_light_status()`
- **SoundManager.h**: `SoundManager::instance()`, play_polygon_sound() 
- **player.h**: `try_and_subtract_player_item()`
- **media.h**: `get_media_data()`
- **items.h**: (item definitions, used indirectly)
- **Packing.h**: `StreamToValue()`, `ValueToStream()` (binary serialization)
- **DamageParser.h**: `Damage_GetParser()`, `Damage_SetPointer()`
- **lua_script.h**: `L_Call_Platform_Activated()` (conditional on `HAVE_LUA`)

Notable: Relies heavily on bitflag macros from platforms.h for state management; uses GetMemberWithBounds for safe indexing; no explicit memory allocation beyond vector growth.
