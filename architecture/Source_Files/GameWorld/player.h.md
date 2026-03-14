# Source_Files/GameWorld/player.h

## File Purpose

Defines the player data structure, physics model, action flags, and game settings for player management in the Aleph One engine. Provides interfaces for player initialization, update, damage handling, and physics calculations including support for networked multiplayer games.

## Core Responsibilities

- Define and manage player state (`player_data` structure) including location, health, weapons, and status
- Configure player physics models and motion (gravity, acceleration, elevation)
- Encode input actions via flag bits (movement, looking, weapon cycling, triggers)
- Track damage given/taken and maintain player statistics
- Provide player settings (energy, oxygen, powerup durations) configurable via XML
- Support teleporting, interlevel transport, and inventory management
- Interface with physics engine for position/velocity updates and collision
- Support action queue buffering for network synchronization (0x400 buffer diameter)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `player_settings_definition` | struct | XML-configurable player globals: initial energy/oxygen, oxygen depletion rates, self-luminosity, visual range |
| `physics_variables` | struct | Per-player physics state: position, velocity, direction, elevation, floor/ceiling heights, step animation phase |
| `player_data` | struct | Complete per-player state: ID, flags, location, suit energy, weapons, powerups, damage history, items, teleport phase |
| `damage_record` | struct | Damage tracking: total damage and kill count |
| `player_shape_definitions` | struct | Animation shape indices: leg poses, torso weapon states, dying animations |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `player_settings` | `player_settings_definition` | global | Global player configuration (energy, oxygen rates, visual arcs, weapon properties) |
| `players` | `player_data*` | global | Array of all player instances in the game |
| `local_player`, `current_player` | `player_data*` | global | Cached pointers to local and current player for fast access |
| `local_player_index`, `current_player_index` | short | global | Indices into players array; use setters to change |
| `team_damage_given`, `team_damage_taken` | `damage_record[8]` | global | Team-wide damage statistics indexed by team color |
| `team_monster_damage_taken`, `team_monster_damage_given` | `damage_record[8]` | global | Monster damage statistics per team |
| `team_friendly_fire` | `damage_record[8]` | global | Friendly fire tracking per team |

## Key Functions / Methods

### initialize_players
- **Purpose:** Initialize all player-related systems at game startup.
- **Calls:** Presumed to call `initialize_player_physics_variables()`, `initialize_player_weapons()`, and action queue setup.

### update_players
- **Signature:** `void update_players(ActionQueues* inActionQueuesToUse, bool inPredictive)`
- **Purpose:** Advance all players one tick (assumes ╬öt = 1 tick); supports predictive updates for network prediction without full state changes.
- **Inputs:** Action queue set to use (allows caller to redirect input), predictive flag.
- **Calls:** `update_player_physics_variables()`, `update_player_weapons()`, damage application.

### new_player
- **Signature:** `short new_player(short team, short color, short player_identifier)`
- **Purpose:** Create and register a new player in the world.
- **Inputs:** Team, color index, unique identifier.
- **Outputs/Return:** Player index (or index into players array).
- **Side effects:** Allocates monster and object slots; updates player count.

### damage_player
- **Signature:** `void damage_player(short monster_index, short aggressor_index, short aggressor_type, struct damage_definition *damage, short projectile_index)`
- **Purpose:** Apply damage to a player from a monster or another player; update damage records.
- **Inputs:** Monster dealing damage, aggressor ID, damage type definition, projectile ID.
- **Side effects:** Reduces suit energy, updates damage history, may trigger death.

### update_player_physics_variables
- **Signature:** `void update_player_physics_variables(short player_index, uint32 action_flags, bool predictive)`
- **Purpose:** Advance a single player's physics state: position, velocity, direction, elevation, and animation phase.
- **Inputs:** Player index, action flags (movement/looking bits), predictive flag.
- **Side effects:** Updates `physics_variables` fields: `position`, `velocity`, `direction`, `elevation`, `step_phase`, floor/ceiling heights.

### mask_in_absolute_positioning_information
- **Signature:** `uint32 mask_in_absolute_positioning_information(uint32 action_flags, _fixed yaw, _fixed pitch, _fixed velocity)`
- **Purpose:** Encode absolute yaw/pitch/velocity data into action flags via bit-packing for network transmission or storage.
- **Inputs:** Base action flags, three fixed-point values.
- **Outputs/Return:** Updated flags with encoded absolute positioning.

### get_player_data
- **Signature:** `player_data *get_player_data(const size_t player_index)`
- **Purpose:** Safe accessor to retrieve a player data structure by index; performs bounds checking.
- **Inputs:** Player index.
- **Outputs/Return:** Pointer to `player_data`, or asserts on invalid index.

### GetRealActionQueues
- **Signature:** `ActionQueues* GetRealActionQueues()`
- **Purpose:** Return the canonical action queue set used by `update_players()` for network and input routing.
- **Outputs/Return:** Pointer to global action queue object (not a modifiable pointer, returned from function to prevent accidental reassignment).

### get_player_shape_definitions
- **Signature:** `player_shape_definitions* get_player_shape_definitions()`
- **Purpose:** Retrieve animation shape indices (leg poses, weapon states) for player rendering.
- **Outputs/Return:** Pointer to global shape definition.

## Control Flow Notes

- **Initialization:** `initialize_players()` ΓåÆ `allocate_player_memory()`, `initialize_player_physics_variables()` per player.
- **Frame update:** `update_world()` (in map.h) calls `update_players(ActionQueues*, bool)` once per tick, which updates physics and weapons.
- **Action input:** Action flags built by input system; passed via action queues; unpacked into yaw/pitch/position via `mask_in_absolute_positioning_information()` and applied in physics update.
- **Damage flow:** Collision/projectile systems call `damage_player()` ΓåÆ suit energy decremented ΓåÆ if energy Γëñ 0, player marked dead via flags.
- **Shutdown:** `delete_player()` removes a player; `mark_player_collections()` unloads rendering data.

**Physics models:** Selectable via `enum` (_editor_model, _earth_gravity_model, _low_gravity_model).

## External Dependencies

- **`cseries.h`:** Common utility types and macros (fixed-point, basic data structures).
- **`world.h`:** World coordinate system (angle, world_distance, point/vector types, trigonometry).
- **`map.h`:** Map structure constants (TICKS_PER_MINUTE, TELEPORTING_DURATION, damage types, game options).
- **`XML_ElementParser.h`:** XML parsing interface for MML configuration.
- **`weapons.h`:** Weapon data structures and constants (for `player_weapon_data`, trigger states).
- **Defined elsewhere:** `ActionQueues` class (action queue management for network sync); monster and object systems (for damage tracking and rendering).

**Macros for flag access** (e.g., `PLAYER_IS_DEAD()`, `SET_PLAYER_ZOMBIE_STATUS()`) provide bit-level manipulation of player state flags defined in `player_data.flags`.
