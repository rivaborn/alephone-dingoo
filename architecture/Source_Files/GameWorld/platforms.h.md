# Source_Files/GameWorld/platforms.h

## File Purpose
Defines platform (moving structure) types, state flags, and data structures for the game world. Provides the interface for creating, updating, and querying platformsΓÇöinteractive geometry like doors, elevators, and crushers that move between discrete positions.

## Core Responsibilities
- Enumerate platform types (doors, platforms with various behaviors)
- Define speed/delay presets for platform movement
- Define static (configuration) and dynamic (runtime) flag sets
- Provide flag testing/setting macros for platform properties
- Define platform data structures for both static config and runtime state
- Declare functions for platform creation, state management, and physics queries
- Provide serialization/deserialization for platform data
- Export XML parser interface for level loading

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `endpoint_owner_data` | struct | Maps platform endpoints to their associated polygon and line indices in the world geometry |
| `static_platform_data` | struct | Immutable configuration: type, speed, delay, height bounds, flags, polygon index, tag (32 bytes) |
| `platform_data` | struct | Runtime state: combines static config, dynamic flags, current floor/ceiling heights, endpoint ownership, parent platform reference (140 bytes) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PlatformList` | `vector<platform_data>` | global (extern) | All active platforms in current map; size determines `MAXIMUM_PLATFORMS_PER_MAP` |

## Key Functions / Methods

### new_platform
- Signature: `short new_platform(struct static_platform_data *data, short polygon_index)`
- Purpose: Create and register a new platform instance from static configuration data
- Inputs: static platform config, polygon index where platform is located
- Outputs/Return: platform index (short)
- Side effects: Adds entry to `PlatformList`; initializes platform state

### update_platforms
- Signature: `void update_platforms(void)`
- Purpose: Main update loop for all platforms; called once per frame
- Side effects: Moves active platforms, checks collisions, applies delays, triggers state changes, activates adjacent platforms

### try_and_change_platform_state
- Signature: `bool try_and_change_platform_state(short platform_index, bool state)`
- Purpose: Attempt to activate (state=true) or deactivate (state=false) a single platform
- Outputs/Return: true if state change succeeded
- Notes: Respects flags like `_platform_cannot_be_externally_deactivated` and `_platform_activates_only_once`

### try_and_change_tagged_platform_states
- Signature: `bool try_and_change_tagged_platform_states(short tag, bool state)`
- Purpose: Activate/deactivate all platforms matching a given tag
- Outputs/Return: true if at least one platform changed
- Side effects: Cascading activation of adjacent/linked platforms

### monster_can_enter_platform / monster_can_leave_platform
- Signature: `short monster_can_enter_platform(short platform_index, short source_polygon_index, world_distance height, world_distance minimum_ledge_delta, world_distance maximum_ledge_delta)`
- Purpose: Determine if AI can traverse onto/off a platform from adjacent geometry
- Outputs/Return: enum indicating accessibility state (never/might/will/is accessible)
- Notes: Considers platform height bounds, ledge height differences, and current platform state

### platform_is_on
- Signature: `bool platform_is_on(short platform_index)`
- Purpose: Query whether a platform is currently active (moving or ready to move)
- Outputs/Return: true if active

### adjust_platform_for_media
- Signature: `void adjust_platform_for_media(short platform_index, bool initialize)`
- Purpose: Update platform heights when water/lava media level changes
- Side effects: Modifies platform floor/ceiling heights; sets media-below flags

### player_touch_platform_state
- Signature: `void player_touch_platform_state(short player_index, short platform_index)`
- Purpose: Handle player interaction when using action key on a platform (e.g., opening a door)

### get_platform_moving_sound
- Signature: `short get_platform_moving_sound(short platform_index)`
- Purpose: Retrieve sound ID for this platform type when moving
- Outputs/Return: sound index

### Platform data packing/unpacking
- Signature: `uint8 *unpack_static_platform_data(uint8 *Stream, static_platform_data *Objects, size_t Count)` (+ 3 variants)
- Purpose: Serialize/deserialize platform data to/from binary streams
- Side effects: Reads/writes binary data; updates stream pointer

### Platforms_GetParser
- Signature: `XML_ElementParser *Platforms_GetParser()`
- Purpose: Return XML parser instance for loading platforms from level files
- Outputs/Return: parser object (defined elsewhere)

**Notes on trivial helpers:** The file defines ~40 flag-testing macros (`PLATFORM_IS_ACTIVE`, `SET_PLATFORM_CAUSES_DAMAGE`, etc.) that wrap `TEST_FLAG16`/`SET_FLAG16` or `TEST_FLAG32`/`SET_FLAG32` calls; these are not individually documented above.

## Control Flow Notes
Platforms are part of the per-frame game world update. Each frame, `update_platforms()` processes all active platforms:
- Moves platforms toward their target height at their configured speed
- Checks for obstruction and adjusts direction if needed
- Fires callbacks when platforms reach discrete levels or fully extend/contract
- Applies delays between movements
- Cascades activation to adjacent platforms if flags permit

Platforms can be activated by players (doors via action key), scripted events (through tags), or other platforms. State changes occur through the `try_and_change_platform_state*` functions, which check flag constraints before allowing transitions.

## External Dependencies
- **XML_ElementParser.h**: Base class for XML parsing; used for level file loading
- **Macro dependencies** (defined elsewhere): `TEST_FLAG16`, `TEST_FLAG32`, `SET_FLAG16`, `SET_FLAG32`, `WORLD_ONE`, `TICKS_PER_SECOND`, `MAXIMUM_VERTICES_PER_POLYGON`
- **Type dependencies** (defined elsewhere): `world_distance`, `int16`, `uint16`, `uint32`, `uint8`
- **Function dependencies**: Flag macros call into a flag-manipulation library (not visible here)
