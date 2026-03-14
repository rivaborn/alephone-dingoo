# Source_Files/GameWorld/monsters.cpp

## File Purpose
Implements monster AI, physics, animation, and lifecycle for the Marathon engine. Handles monster spawning, pathfinding, target acquisition, combat behavior, damage application, and serialization. Core to the enemy entity system.

## Core Responsibilities
- Monster creation with difficulty-based scaling (promotion/demotion) and map placement
- Frame-by-frame monster update loop: animation, pathfinding, target acquisition, physics, state transitions
- Pathfinding and movement with cost-based navigation and terrain interaction (platforms, doors, media)
- Target acquisition via line-of-sight checking and attitude determination (friendly/hostile/neutral)
- Behavioral modes (locked, losing lock, lost lock, unlocked) and action states (moving, attacking, dying, teleporting)
- Physics simulation including vertical motion, external velocity from impacts, and gravity
- Damage application, death handling, shrapnel effects, and monster-monster interaction
- Save/load serialization via pack/unpack functions
- Runtime XML configuration for damage kick (knockback) definitions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| monster_pathfinding_data | struct | Context for pathfinding cost calculations; holds monster definition, data, and zone-crossing flags |
| damage_kick_definition | struct | Knockback physics: base velocity value, multiplier by damage amount, vertical component flag |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| IntersectedObjects | vector\<short\> | static | Collision check buffer for pathfinding; growable list of nearby object indices |
| damage_kick_definitions | damage_kick_definition[24] | global | Knockback parameters per damage type; indexed by `_damage_*` enum |
| original_damage_kick_definitions | pointer | static | Backup for XML-based runtime overrides of damage kicks |
| DamageKickParser | XML_DamageKickParser | static | XML element parser for individual damage kick entries |
| DamageKicksParser | XML_ElementParser | static | Root XML parser for damage_kicks container |

## Key Functions / Methods

### new_monster
- **Signature:** `short new_monster(object_location *location, short monster_type)`
- **Purpose:** Create a new monster at a map location, handling difficulty-based spawn filtering, promotion/demotion, and initialization.
- **Inputs:** Location (world position, polygon, flags); monster type enum.
- **Outputs/Return:** Monster index on success; NONE on failure.
- **Side effects:** Allocates monster slot, creates object, marks collections for loading, updates object frequency counts.
- **Calls:** `get_monster_definition()`, `new_map_object()`, `nearest_goal_polygon_index()`, `object_was_just_added()`.
- **Notes:** Difficulty affects spawn rate (wuss/easy drop monsters; major/carnage promote minors). Blind/deaf/float flags copied from map object. Demoted monsters are slightly weaker variants.

### move_monsters
- **Signature:** `void move_monsters(void)`
- **Purpose:** Update all monsters each frame; main entry point for monster simulation.
- **Inputs:** None (operates on global monster array).
- **Outputs/Return:** None.
- **Side effects:** Updates monster state (animation, position, action, mode), modifies object positions, may spawn effects, may change monster state to dead/inactive.
- **Calls:** `cause_polygon_damage()`, `update_monster_vertical_physics_model()`, `animate_object()`, `find_closest_appropriate_target()`, `change_monster_target()`, `clear_line_of_sight()`, `generate_new_path_for_monster()`, `handle_moving_or_stationary_monster()`, `execute_monster_attack()`, `try_monster_attack()`, `set_monster_action()`, `monster_needs_path()`, `update_monster_physics_model()`, `cause_shrapnel_damage()`, `kill_monster()`, `activate_nearby_monsters()`.
- **Notes:** Uses budgeting flags (`last_monster_index_to_get_time`, `last_monster_index_to_build_path`) to spread expensive pathfinding across frames. Civilian kill counter decremented periodically.

### monster_died
- **Signature:** `void monster_died(short target_index)`
- **Purpose:** Cleanup when a monster dies; orphan projectiles and redirect locked-on monsters.
- **Inputs:** Target monster index.
- **Outputs/Return:** None.
- **Side effects:** Clears paths, marks unlocked, orphans projectiles; updates all monsters locked on target to find new targets.
- **Calls:** `orphan_projectiles()`, `delete_path()`, `find_closest_appropriate_target()`, `play_object_sound()`, `change_monster_target()`, `set_monster_action()`, `set_monster_mode()`.
- **Notes:** Called before target is removed from array. Plays kill sound for locked-on monsters.

### initialize_monsters
- **Signature:** `void initialize_monsters(void)`
- **Purpose:** Initialize monster subsystem state at game start.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Resets civilian kill count, pathfinding budget counters, random seeds.
- **Calls:** None directly.

### get_monster_data
- **Signature:** `monster_data *get_monster_data(short monster_index)`
- **Purpose:** Accessor with bounds and slot-usage validation.
- **Inputs:** Monster index.
- **Outputs/Return:** Pointer to monster_data.
- **Side effects:** Asserts on invalid index or free slot.
- **Notes:** Validates using SLOT_IS_USED macro; used throughout to ensure safe access.

### damage_monsters_in_radius
- **Signature:** `void damage_monsters_in_radius(short primary_target_index, short aggressor_index, short aggressor_type, world_point3d *epicenter, short epicenter_polygon_index, world_distance radius, damage_definition *damage, short projectile_index)`
- **Purpose:** Apply damage to all monsters within a radius (explosions, etc.).
- **Inputs:** Target, attacker indices/type; damage epicenter; radius; damage definition; projectile index.
- **Outputs/Return:** None.
- **Side effects:** Damages all monsters in radius, may apply knockback, may kill monsters.
- **Calls:** `damage_monster()` for each monster in range.

### monster_pathfinding_cost_function
- **Signature:** `int32 monster_pathfinding_cost_function(short source_polygon, short line_index, short destination_polygon, void *data)`
- **Purpose:** Cost evaluator for A\* pathfinding; discourages overcrowding, penalizes height changes, blocks impassable areas.
- **Inputs:** Source/destination polygon indices; connecting line; user data (monster_pathfinding_data).
- **Outputs/Return:** Cost (positive), or -1 to block path.
- **Side effects:** None.
- **Calls:** `get_polygon_data()`, `get_line_data()`, `get_object_data()`, `monster_can_enter_platform()`, `monster_can_leave_platform()`, `get_media_data()`, `IsMediaDangerous()`.
- **Notes:** Blocks solid lines, too-narrow passages, zones if crossing disabled. Penalizes crowds, media, height changes. Platform transitions respect monster dimensions.

### find_obstructing_terrain_feature
- **Signature:** `static short find_obstructing_terrain_feature(short monster_index, short *feature_index, short *relevant_polygon_index)`
- **Purpose:** Detect doors, platforms, hazards (sniper ledges, media) blocking monster path.
- **Inputs:** Monster index.
- **Outputs/Return:** Feature type enum; outputs feature index and relevant polygon.
- **Side effects:** May modify monster's desired_height.
- **Calls:** `ray_to_line_segment()`, `find_line_crossed_leaving_polygon()`, `find_adjacent_polygon()`, `get_polygon_data()`, `get_media_data()`, `IsMediaDangerous()`.
- **Notes:** Handles platform entering/leaving, floating/flying transitions, media wading, sniper ledges.

### position_monster_projectile
- **Signature:** `static short position_monster_projectile(short aggressor_index, short target_index, attack_definition *attack, world_point3d *origin, world_point3d *destination, world_point3d *_vector, angle theta)`
- **Purpose:** Calculate origin, destination, and direction vector for monster projectile fire.
- **Inputs:** Attacker/target indices; attack definition; theta angle; output buffers.
- **Outputs/Return:** Polygon index of projectile origin; fills buffers with calculated vectors.
- **Side effects:** Updates aggressor's elevation field.
- **Calls:** `get_monster_data()`, `get_object_data()`, `get_monster_dimensions()`, `find_new_object_polygon()`.
- **Notes:** Aims at 3/4 height of target. Calculates elevation angle if target given.

### update_monster_physics_model
- **Signature:** `static void update_monster_physics_model(short monster_index)`
- **Purpose:** Apply gravity, external velocity (from knockback), and vertical motion constraints.
- **Inputs:** Monster index.
- **Outputs/Return:** None.
- **Side effects:** Updates monster's location and vertical_velocity.
- **Calls:** `get_monster_data()`, `get_object_data()`, `get_polygon_data()`, `translate_monster()`.
- **Notes:** Handles terminal velocity, floor contact, ceiling collision.

### unpack_monster_data / pack_monster_data
- **Signature:** `uint8 *unpack_monster_data(uint8 *Stream, monster_data *Objects, size_t Count)` / `pack` variant
- **Purpose:** Serialize/deserialize monster state for save/load.
- **Inputs:** Byte stream, object array, count.
- **Outputs/Return:** Advanced stream pointer.
- **Side effects:** Reads/writes all monster_data fields.
- **Calls:** `StreamToValue()` / `ValueToStream()` macros.
- **Notes:** Asserts correct size at end; padding skipped.

### unpack_monster_definition / pack_monster_definition
- **Signature:** `uint8 *unpack_monster_definition(uint8 *Stream, monster_definition *Objects, size_t Count)` / `pack` variant
- **Purpose:** Serialize monster type templates (vitality, sounds, shapes, attacks, physics).
- **Inputs:** Stream, definition array, count.
- **Outputs/Return:** Advanced stream pointer.
- **Calls:** `StreamToValue()` / `ValueToStream()`, `unpack_damage_definition()` / `pack_damage_definition()`.

**Smaller utility functions:** `clear_line_of_sight()`, `find_closest_appropriate_target()`, `change_monster_target()`, `get_monster_attitude()`, `execute_monster_attack()`, `try_monster_attack()`, `handle_moving_or_stationary_monster()`, `activate_monster()`, `deactivate_monster()`, `load_monster_sounds()`, `nearest_goal_polygon_index()`, `nearest_goal_cost_function()` are documented in prototypes but handle target logic, attack triggering, mode/action state management.

## Control Flow Notes
**Init Phase:** `initialize_monsters()` resets global state on game start. `initialize_monsters_for_new_level()` invalidates cached paths when a map loads.

**Spawn Phase:** `new_monster()` creates instances during map initialization (via placement system).

**Update Phase (per frame):** `move_monsters()` is the main loop. It iterates all monsters:
- **Active monsters:** Apply damage effects ΓåÆ update animation ΓåÆ acquire/reacquire targets ΓåÆ generate paths (budgeted) ΓåÆ execute action state machine (moving ΓåÆ attacking ΓåÆ dying ΓåÆ teleporting, etc.) ΓåÆ apply physics.
- **Inactive monsters:** Limited target search for activation triggers.

**Cleanup:** `monster_died()` notifies locked-on monsters when target expires. `remove_monster()` deallocates a monster.

## External Dependencies
- **Map system:** `get_polygon_data()`, `get_line_data()`, `find_adjacent_polygon()`, `cause_polygon_damage()`, `find_new_object_polygon()`, `ray_to_line_segment()`, `flood_map()` (pathfinding).
- **Object/Rendering:** `get_object_data()`, `new_map_object()`, `remove_map_object()`, `animate_object()`, `GET_OBJECT_ANIMATION_FLAGS()`, `GET_SEQUENCE_FRAME()`.
- **Physics/Platforms:** `monster_can_enter_platform()`, `monster_can_leave_platform()`, `get_media_data()`, `IsMediaDangerous()`.
- **Projectiles:** `fire_projectile()`, `position_monster_projectile()` (custom positioning), `orphan_projectiles()`.
- **Effects:** `new_effect()`, `teleport_object_in/out()`.
- **Sound:** `SoundManager::instance()->LoadSound()`, `play_object_sound()`.
- **Lua:** `L_Invalidate_Monster()` (optional scripting hook).
- **Monster Definitions:** Included via `monster_definitions.h` (external static data).
- **XML Parsing:** `XML_ElementParser` base class, `XML_DamageKickParser` for config.
