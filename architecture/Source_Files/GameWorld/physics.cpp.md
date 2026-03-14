# Source_Files/GameWorld/physics.cpp

## File Purpose
Core player physics simulation engine for Marathon. Manages position, velocity, rotation, collision detection, and environmental interactions (media, gravity, jumping). Implements a continuous lag-tolerant networked physics model with frame-by-frame deterministic updates.

## Core Responsibilities
- Initialize and update player physics state each frame
- Calculate player movement based on input action flags (forward, strafe, rotate, look)
- Simulate gravity, jumping, and external forces (acceleration, impacts)
- Resolve collisions with walls, platforms, and solid objects
- Track player position and orientation (heading, pitch, elevation)
- Manage media (liquid) interaction and buoyancy
- Update animation state (step phase/amplitude for bob)
- Shadow physics state into world objects and camera position
- Pack/unpack physics constants for network sync and save/load

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `physics_variables` | struct | Player movement state: position, velocity, angles, flags, heights |
| `physics_constants` | struct | Physics parameters (max velocities, accelerations, gravity) |
| `fixed_point3d` | typedef | 3D position in fixed-point coordinates |
| `fixed_vector3d` | typedef | 3D velocity/acceleration vector |
| `player_data` | struct | Complete player state (defined in player.h) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `saved_points`, `saved_thetas` | `world_point3d*`, `angle*` | static | Divergence checking for network sync verification (DEBUG only) |
| `saved_point_count`, `saved_point_iterations` | `short`, `static short` | static | Counter for divergence test data |
| `physics_models` | `physics_constants[]` | extern | Array of physics parameter sets (walking/running) |
| `local_player` | `player_data*` | extern | Current local player reference |
| `static_world` | `static_data*` | extern | Map environment flags and physics model selection |

## Key Functions / Methods

### initialize_player_physics_variables
- **Signature:** `void initialize_player_physics_variables(short player_index)`
- **Purpose:** Initialize physics state when player enters a level or respawns
- **Inputs:** Player index into global players array
- **Outputs/Return:** None (modifies player's variables via side effect)
- **Side effects:** Zeros physics_variables struct, sets up divergence tracking, calls `instantiate_physics_variables` to sync with world
- **Calls:** `get_player_data()`, `get_monster_data()`, `get_object_data()`, `get_physics_constants_for_model()`, `instantiate_physics_variables()`
- **Notes:** Assumes player's object exists in monsters array; `obj_set()` fills struct with 0x80 pattern (DEBUG check for uninitialized memory)

### update_player_physics_variables
- **Signature:** `void update_player_physics_variables(short player_index, uint32 action_flags, bool predictive)`
- **Purpose:** Main per-frame physics update; apply input, gravity, collisions
- **Inputs:** Player index, action flags (movement/turning input), predictive flag (true = reduced state modification for save/restore)
- **Outputs/Return:** None (modifies player variables)
- **Side effects:** Updates position, velocity, angles; calls collision detection; emits divergence warnings
- **Calls:** `physics_update()`, `instantiate_physics_variables()`
- **Notes:** Predictive mode reduces side effects for prediction/rollback systems; divergence check logs network sync mismatches

### physics_update
- **Signature:** `static void physics_update(struct physics_constants *constants, struct physics_variables *variables, struct player_data *player, uint32 action_flags)`
- **Purpose:** Core physics simulation loop; calculate new position, velocity, and angles
- **Inputs:** Physics parameters, current state variables, player (for dead-player checks), action flags
- **Outputs/Return:** None (modifies variables in-place)
- **Side effects:** Modifies velocity, position, angles, flags; absorbs bounces; updates step animation
- **Calls:** Inline math (sine/cosine tables, bit shifts), conditional checks on flags
- **Notes:** Dead players lose input control; bouncing uses `COEFFICIENT_OF_ABSORBTION`; recentering auto-corrects pitch when running straight; swimming applies lift; step phase drives bob animation

### instantiate_physics_variables
- **Signature:** `static void instantiate_physics_variables(struct physics_constants *constants, struct physics_variables *variables, short player_index, bool first_time, bool take_action)`
- **Purpose:** Apply physics results to world: collision detection, camera positioning, shadow player location into world objects
- **Inputs:** Physics constants, calculated variables, player index, first-time flag (skip some updates), take_action (emit side effects like `monster_moved()`)
- **Outputs/Return:** None (modifies world state)
- **Side effects:** Calls `keep_line_segment_out_of_walls()`, `translate_map_object()`, updates player.camera_location, player.location, monster sound location, media flags
- **Calls:** `get_player_data()`, `get_monster_data()`, `get_object_data()`, `get_polygon_data()`, `get_media_data()`, `keep_line_segment_out_of_walls()`, `translate_map_object()`, `changed_polygon()`, `monster_moved()`
- **Notes:** Applies DROP_DEAD_HEIGHT offset for dead players (zero if chase-cam active); disables step bob when chase-cam active; calls `changed_polygon()` on entry/exit for triggers

### accelerate_player
- **Signature:** `void accelerate_player(short monster_index, world_distance vertical_velocity, angle direction, world_distance velocity)`
- **Purpose:** Apply external forces (explosions, pushback) to player velocity
- **Inputs:** Monster (player) index, vertical impulse, direction, horizontal magnitude
- **Outputs/Return:** None
- **Side effects:** Adds to external_velocity, clamps to terminal velocity
- **Calls:** Lookup cosine/sine tables
- **Notes:** Used by damage/explosions; velocity is clamped with `PIN()`

### get_physics_constants_for_model
- **Signature:** `struct physics_constants *get_physics_constants_for_model(short physics_model, uint32 action_flags)`
- **Purpose:** Fetch physics parameter set based on map physics model and run state
- **Inputs:** Physics model enum (_editor_model, _earth_gravity_model, _low_gravity_model), action flags (check _run_dont_walk)
- **Outputs/Return:** Pointer to appropriate physics_constants in global physics_models array
- **Side effects:** None
- **Calls:** None (lookup table)
- **Notes:** Low-gravity and other models trigger asserts (not implemented); returns walking or running variant

### unpack_physics_constants / pack_physics_constants
- **Signature:** `uint8 *unpack_physics_constants(uint8 *Stream, physics_constants *Objects, size_t Count)` and pack variant
- **Purpose:** Serialize/deserialize physics parameters for network/save files
- **Inputs:** Byte stream pointer, object array, count
- **Outputs/Return:** Updated stream pointer (after read/write)
- **Side effects:** Modifies stream and objects in-place
- **Calls:** `StreamToValue()` / `ValueToStream()` macros
- **Notes:** Processes all physics_constants fields in fixed order; asserts final size matches expected `SIZEOF_physics_constants`

## Control Flow Notes
- **Frame tick sequence:** `update_player_physics_variables()` ΓåÆ `physics_update()` ΓåÆ `instantiate_physics_variables()`. 
- **Initialization:** `initialize_player_physics_variables()` called once per respawn/level entry, then `update_player_physics_variables()` every tick.
- **Dead player behavior:** Dead players auto-orient based on momentum (dot product of velocity and facing); cannot move via input but still fall/bounce.
- **Media interaction:** Flags `_FEET_BELOW_MEDIA_BIT` / `_HEAD_BELOW_MEDIA_BIT` are set based on media height; affect gravity, terminal velocity, and swimming.
- **Predictive mode:** When `predictive=true`, `instantiate_physics_variables()` skips side effects (`take_action=false`) for rollback-safe state snapshots.

## External Dependencies
- **Key includes:** `render.h` (view data), `map.h` (polygon/line/object queries), `player.h` (player_data, physics_variables), `interface.h` (game state), `monsters.h` (monster_data), `media.h` (media_data), `ChaseCam.h` (chase-cam state check)
- **Key externals:** `static_world` (map physics model, environment flags), `local_player` (current player), `physics_models` (from physics_models.h), trigonometry tables (`cosine_table[]`, `sine_table[]`), `get_physics_constants_for_model()` (also defined locally)
- **Collision/world functions:** `keep_line_segment_out_of_walls()`, `translate_map_object()`, `legal_player_move()`, `find_new_object_polygon()` (defined in map.c)
- **Helper macros:** `PLAYER_IS_DEAD()`, `WORLD_TO_FIXED()`, `FIXED_TO_WORLD()`, `PIN()`, `SGN()`, bit manipulation (`FIXED_FRACTIONAL_BITS`, angle normalization)
