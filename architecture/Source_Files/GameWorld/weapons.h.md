# Source_Files/GameWorld/weapons.h

## File Purpose
Header file defining the weapon system for a game engine (Aleph One). Declares data structures for weapon state per player and per trigger, enumerations for weapon types and actions, and function prototypes for weapon initialization, updates, serialization, and queries.

## Core Responsibilities
- Define weapon type identifiers and weapon action states (idle, charging, firing)
- Model weapon state: per-player inventory, per-weapon trigger state (ammo, fire rate), and shell casing physics
- Manage weapon display/rendering information (positioning, animation frames, visual modes)
- Initialize and update weapon systems at game startup, new game, and per-frame
- Handle ammunition tracking and reloading when items are picked up
- Support serialization (pack/unpack) for save/load and network sync
- Provide queries for UI/display (current weapon, ammo, firing state)
- Integrate XML-based weapon configuration

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `weapon_display_information` | struct | Rendering data for 3D weapon models: collection/shape indices, screen position (low/center/high), animation frames/phases, flipping, transfer mode |
| `trigger_data` | struct | Per-trigger state: fire phase, ammo loaded/fired/hit, shell casing timing, animation sequence, firing duration |
| `weapon_data` | struct | Single weapon state: type ID, flags, both primary+secondary triggers |
| `shell_casing_data` | struct | Physics state of ejected casings: type, frame, flags, position (x,y), velocity (vx,vy) |
| `player_weapon_data` | struct | Complete per-player weapons: current weapon, desired weapon, all weapons array, shell casings array |
| Weapon enum | enum | 11 weapon types: fist, pistol, plasma pistol, assault rifle, missile launcher, flamethrower, alien shotgun, shotgun, ball, SMG; plus pseudo-weapons (dual-wield variants) |
| Weapon action enum | enum | Animation states: idle, charging, firing |
| Trigger enum | enum | Primary and secondary triggers |
| Display position enum | enum | Low/center/high positioning modes for screen space |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_weapon_manager
- Signature: `void initialize_weapon_manager(void)`
- Purpose: One-time engine startup for weapon subsystem
- Inputs: None
- Outputs: None
- Side effects: Initializes global weapon state/definitions
- Calls: Not inferable from header

### initialize_player_weapons_for_new_game / initialize_player_weapons
- Signature: `void initialize_player_weapons_for_new_game(short player_index)` and `void initialize_player_weapons(short player_index)`
- Purpose: Reset weapon inventory for new game or when player created
- Inputs: Player index
- Outputs: None
- Side effects: Resets ammo, weapons array, trigger states
- Notes: First function is for new-game scenarios; second for per-player post-creation setup

### update_player_weapons
- Signature: `void update_player_weapons(short player_index, uint32 action_flags)`
- Purpose: Per-frame weapon state update (fire, reload, animations)
- Inputs: Player index, action flags (fire primary/secondary, etc.)
- Outputs: None
- Side effects: Updates trigger state, ammo, animation frames, shell casing physics
- Calls: Not inferable from header

### get_weapon_display_information
- Signature: `bool get_weapon_display_information(short *count, struct weapon_display_information *data)`
- Purpose: Retrieve display info for current weapon frame (called repeatedly until false)
- Inputs: Count pointer (incremented), display data pointer (filled)
- Outputs: Boolean (true = more data, false = done)
- Side effects: None
- Notes: Iterator pattern for rendering

### player_hit_target
- Signature: `void player_hit_target(short player_index, short weapon_identifier)`
- Purpose: Notify weapon system of projectile impact (feedback, ammo accounting)
- Inputs: Player index, weapon identifier from projectile
- Outputs: None
- Side effects: Updates shots_hit counter, may affect weapon/trigger state

### Ammo/state queries
- `get_player_weapon_ammo_count()`, `get_player_weapon_ammo_maximum()`, `get_player_weapon_ammo_type()`, `get_player_weapon_drawn()`, `get_player_desired_weapon()`, `get_player_weapon_mode_and_type()`
- Purpose: UI/display and logic queries for weapon state
- Notes: Trivial accessors; critical for UI updates and drawing

### Serialization routines
- `pack_player_weapon_data()`, `unpack_player_weapon_data()`, `pack_weapon_definition()`, `unpack_weapon_definition()`
- Purpose: Binary serialization for save/load and network transmission
- Inputs/Outputs: Byte stream, count
- Notes: Size constants provided: `SIZEOF_weapon_definition = 134`, `SIZEOF_player_weapon_data = 472`

### Other entry points
- `process_new_item_for_reloading()` ΓÇö ammo pickup handling
- `discharge_charged_weapons()` ΓÇö cleanup on player death
- `check_player_weapons_for_environment_change()` ΓÇö level-change weapon validity
- `mark_weapon_collections()` ΓÇö asset load/unload
- `init_weapon_definitions()`, `get_number_of_weapon_types()`, `Weapons_GetParser()` ΓÇö definition/config setup

## Control Flow Notes
**Initialization phase**: `initialize_weapon_manager()` (once), then `initialize_player_weapons_for_new_game()` per player.

**Per-frame**: `update_player_weapons()` drives firing, reload, and animation logic. `get_weapon_display_information()` iteratively feeds rendering data.

**Event-driven**: `process_new_item_for_reloading()` on pickup, `player_hit_target()` on projectile impact, `discharge_charged_weapons()` on death, `check_player_weapons_for_environment_change()` on level load.

**Queries**: `get_player_weapon_*()` called by UI/draw code for display.

**Serialization**: `pack/unpack_*` called by save/load/network systems.

## External Dependencies
- **Custom typedefs**: `_fixed` (fixed-point), `uint16`, `uint8`, `int16`, `int32`, `size_t`, `bool`
- **XML support**: References `XML_ElementParser` (defined elsewhere; used for weapon configuration parsing)
- **Legacy API**: `get_weapon_array()`, `calculate_weapon_array_length()` marked as superseded by pack/unpack routines
- **No direct #include directives shown** ΓÇö definitions of `_fixed`, `uint*`, etc. provided by project header chain
- Comments reference `player.c`, `lua_script.cpp` (consumers of these structures)
