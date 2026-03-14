# Source_Files/GameWorld/projectiles.cpp

## File Purpose
Implements projectile physics simulation, collision detection, and lifecycle management for the Marathon/Aleph One game engine. Handles creation, movement, impact detection, and destruction of projectiles with support for gravity, bouncing, media penetration, guided targeting, and damage-on-impact.

## Core Responsibilities
- Projectile instantiation and initialization with velocity/elevation
- Per-tick physics simulation (gravity, wander, guided steering)
- Collision detection against terrain, monsters, scenery, and fluid surfaces
- Media boundary penetration handling (PMB flag special case)
- Damage application (direct hit or area-of-effect)
- Detonation effects and contrail generation
- Resource loading and serialization (packing/unpacking)
- Projectile cleanup on removal or monster owner death (orphaning)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| projectile_data | struct | Runtime state: type, position (via object), velocity, gravity, elevation, owner, target, flags |
| projectile_definition | struct | Static definition: speed, radius, damage, effects, flags, audio |
| IntersectedObjects | static vector\<short\> | Collision buffer; reused per translate_projectile call |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| IntersectedObjects | vector\<short\> | static | Growable list of object indices to check for collisions during translate_projectile |
| alien_projectile_override | short | global | NONE or projectile type; replaces alien projectile type if set |
| human_projectile_override | short | global | NONE or projectile type; replaces human projectile type if set |
| projectiles | projectile_data[] | global (extern) | Array of all active projectiles |
| projectile_definitions | projectile_definition[] | global (extern) | Array of projectile type definitions |

## Key Functions / Methods

### new_projectile
- **Signature:** `short new_projectile(world_point3d *origin, short polygon_index, world_point3d *_vector, angle delta_theta, short type, short owner_index, short owner_type, short intended_target_index, _fixed damage_scale)`
- **Purpose:** Allocate and initialize a new projectile instance in the world.
- **Inputs:** Origin location/polygon; velocity vector; firing angle error (delta_theta); projectile type; owner (monster/player index + type); intended target for guided projectiles; damage multiplier.
- **Outputs/Return:** Projectile index on success, NONE on failure (no free slots or object creation failed).
- **Side effects:** Allocates map object (visual shape), marks projectile slot used, modifies IntersectedObjects if preflight is triggered.
- **Calls:** `adjust_projectile_type()`, `get_projectile_definition()`, `new_map_object3d()`, `animate_object()`.
- **Notes:** Applies delta_theta noise to facing/elevation unless flags forbid it. Respects alien/human projectile overrides. Elevation and distance_travelled initialized to 0.

### move_projectiles
- **Signature:** `void move_projectiles(void)`
- **Purpose:** Main per-tick projectile simulation loop; updates all active projectiles.
- **Inputs:** None (uses global `projectiles` array and `dynamic_world`).
- **Outputs/Return:** None (modifies world state in place).
- **Side effects:** Updates object locations, creates effects, damages monsters/scenery, removes projectiles on detonation or max range, calls Lua callbacks.
- **Calls:** `get_projectile_definition()`, `get_object_data()`, `animate_object()`, `update_guided_projectile()`, `translate_projectile()`, `damage_scenery()`, `damage_monster()`, `damage_monsters_in_radius()`, `remove_projectile()`, `new_effect()`, `L_Call_Projectile_Detonated()`, `translate_map_object()`.
- **Notes:** Gravity applied based on definition flags (_affected_by_gravity, etc.). Guided projectiles updated every other tick. Bouncing checks minimum rebound velocity. Media boundary penetration (PMB flag) prevents removal even on impact. Contrails spawned at configurable intervals (max count limit). Flyby sounds triggered for current player. Max range enforced by tracking distance_travelled.

### translate_projectile
- **Signature:** `uint16 translate_projectile(short type, world_point3d *old_location, short old_polygon_index, world_point3d *new_location, short *new_polygon_index, short owner_index, short *obstruction_index, short *last_line_index, bool preflight)`
- **Purpose:** Detect and report collisions along a projectile path; clip new_location to impact point.
- **Inputs:** Projectile type; old/new positions and polygon; owner (to exclude self-collision); preflight flag (validation mode).
- **Outputs/Return:** Flags indicating what was hit (_projectile_hit, _projectile_hit_monster, _projectile_hit_floor, _projectile_hit_media, _projectile_hit_scenery, _projectile_hit_landscape, _flyby_of_current_player); modifies new_location, new_polygon_index, obstruction_index.
- **Side effects:** Clears and fills IntersectedObjects; may call `try_and_toggle_control_panel()` on control panel lines.
- **Calls:** `get_projectile_definition()`, `get_polygon_data()`, `get_media_data()`, `possible_intersecting_monsters()`, `find_line_crossed_leaving_polygon()`, `get_line_data()`, `find_line_intersection()`, `find_adjacent_polygon()`, `find_floor_or_ceiling_intersection()`, `get_object_data()`, `distance2d()`, `get_monster_dimensions()`, `get_scenery_dimensions()`, `MONSTER_IS_PLAYER()`, `monster_index_to_player_index()`.
- **Notes:** Implements "penetrates media" and "penetrates media boundary" (PMB) logic; PMB allows projectile to surface-detonation-and-continue. Handles transparent walls (line.flags). Performs line-polygon crossing tests and object-radius collision checks. Long-distance friendly (uses int32 for large distance calculations). Returns clipped new_location at first collision point.

### preflight_projectile
- **Signature:** `bool preflight_projectile(world_point3d *origin, short origin_polygon_index, world_point3d *destination, angle delta_theta, short type, short owner, short owner_type, short *obstruction_index)`
- **Purpose:** Validate that a projectile can be fired from a location without spawning inside geometry or media.
- **Inputs:** Origin; destination (to calculate elevation); projectile type; owner info; obstruction_index output.
- **Outputs/Return:** true if legal, false otherwise. Sets obstruction_index to first monster hit (or NONE).
- **Side effects:** Calls `translate_projectile()` in preflight mode.
- **Calls:** `get_projectile_definition()`, `get_polygon_data()`, `get_media_data()`, `translate_projectile()`, `get_object_data()`.
- **Notes:** Checks elevation bounds (must not be too vertical). Checks origin is above media surface. Respects _penetrates_media flag.

### update_guided_projectile
- **Signature:** `static void update_guided_projectile(short projectile_index)`
- **Purpose:** Steer a guided projectile toward its target; called once per tick if target valid and tick count even.
- **Inputs:** Projectile index; target obtained from projectile.target_index.
- **Outputs/Return:** None (modifies projectile's facing and elevation).
- **Side effects:** Updates projectile_object->facing and projectile->elevation.
- **Calls:** `get_projectile_data()`, `get_monster_data()`, `get_object_data()`, `get_monster_dimensions()`.
- **Notes:** Loses lock on invisible targets unless on _total_carnage_level. Uses signed angle deltas (clamped to MAX_GUIDED_DELTA_YAW/PITCH). Difficulty level adjusts turn speed.

### detonate_projectile
- **Signature:** `void detonate_projectile(world_point3d *origin, short polygon_index, short type, short owner_index, short owner_type, _fixed damage_scale)`
- **Purpose:** Create area-of-effect damage and detonation effects at impact location.
- **Inputs:** Impact location/polygon; projectile type; owner info; damage multiplier.
- **Outputs/Return:** None.
- **Side effects:** Damages all monsters in area; spawns detonation effect; calls Lua callback.
- **Calls:** `get_projectile_definition()`, `damage_monsters_in_radius()`, `new_effect()`, `L_Call_Projectile_Detonated()`.
- **Notes:** Only meaningful if definition.area_of_effect > 0.

### remove_projectile
- **Signature:** `void remove_projectile(short projectile_index)`
- **Purpose:** Deallocate a projectile and remove its visual object from the world.
- **Inputs:** Projectile index.
- **Outputs/Return:** None.
- **Side effects:** Marks slot free, removes map object, invalidates in Lua.
- **Calls:** `get_projectile_data()`, `remove_map_object()`, `L_Invalidate_Projectile()`.

### adjust_projectile_type
- **Signature:** `static short adjust_projectile_type(world_point3d *origin, short polygon_index, short type, short owner_index, short owner_type, short intended_target_index, _fixed damage_scale)`
- **Purpose:** Promote projectile type if fired from underwater/media (e.g., staff ΓåÆ staff bolt).
- **Inputs:** Origin; polygon; projectile type; owner/target (unused).
- **Outputs/Return:** Same or promoted projectile type.
- **Side effects:** None.
- **Calls:** `get_polygon_data()`, `get_media_data()`, `get_projectile_definition()`.

### ProjectileIsGuided
- **Signature:** `bool ProjectileIsGuided(short Type)`
- **Purpose:** Query whether a projectile type has the _guided flag set.
- **Inputs:** Projectile type.
- **Outputs/Return:** true if guided.
- **Calls:** `get_projectile_definition()`.

### Packing/Unpacking Functions
- `unpack_projectile_data()`, `pack_projectile_data()`, `unpack_projectile_definition()`, `pack_projectile_definition()`
- **Purpose:** Serialize/deserialize projectile state and definitions to/from byte streams.
- **Inputs/Outputs:** Stream pointer, object array, count.
- **Notes:** Use StreamToValue/ValueToStream macros. Assert on size mismatch.

## Control Flow Notes
**Initialization:** `init_projectile_definitions()` copies original definitions.

**Per-Frame:** `move_projectiles()` is called once per game tick to update all active projectiles. Guided projectiles are steered every other tick. Collisions are checked via `translate_projectile()`, which may trigger impact effects, damage, and removal or continuation (if PMB flag set).

**Impact:** On collision, `move_projectiles()` either calls `detonate_projectile()` for area damage or applies direct damage via `damage_monster()`. Persistent projectiles mark themselves as "caused damage" and remain; others are removed unless PMB-flagged (media boundary penetration).

**Cleanup:** `remove_all_projectiles()` empties the map on level change. `orphan_projectiles()` reassigns owner to NONE when owner dies to prevent dangling references.

## External Dependencies
- **Notable includes:** cseries.h (memory, macros), map.h (polygon/object access), effects.h (effect spawning), monsters.h (monster damage), player.h (player indexing), scenery.h (scenery damage), media.h (media data), SoundManager.h (audio), items.h (item management), projectile_definitions.h (definition array), dynamic_limits.h (max object counts), Packing.h (serialization), lua_script.h (Lua callbacks)
- **Defined elsewhere:** `projectiles`, `projectile_definitions`, all polygon/object/monster data accessors, `dynamic_world`, `current_player_index`, `temporary` (debug buffer), global_random(), sine_table/cosine_table, various distance/intersection math functions
