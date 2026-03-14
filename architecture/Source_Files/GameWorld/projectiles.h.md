# Source_Files/GameWorld/projectiles.h

## File Purpose
Header file for the projectile subsystem in the Marathon/Aleph One game engine. Defines the projectile data structure, enumeration of 39 projectile types (weapons, effects, enemy attacks), and provides functions for creating, updating, colliding, and managing active projectiles in the game world.

## Core Responsibilities
- Define enumeration of projectile types (rockets, grenades, bullets, energy bolts, fusion dispersals, drain effects, etc.)
- Maintain and expose the global `ProjectileList` vector of active projectiles
- Provide CRUD operations for projectiles: creation, movement, removal, and cleanup
- Handle projectile collision detection and detonation via `translate_projectile()`
- Manage projectile owner relationships (track owner index/type, handle orphaning when owner dies)
- Provide serialization (pack/unpack) for projectile data and definitions
- Query projectile attributes (e.g., `ProjectileIsGuided()`)
- Support contrail/trail effects and flyby sound cues

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `projectile_data` | struct (32 bytes) | Runtime state of an active projectile: type, object_index, target, elevation, owner, flags, contrail tracking, distance, gravity, damage scale, permutation |
| Projectile type enum | enum | Enumeration of 39 projectile types: `_projectile_rocket`, `_projectile_grenade`, `_projectile_staff`, `_projectile_fusion_bolt_minor/major`, `_projectile_hunter`, `_projectile_smg_bullet`, etc. |
| `translate_projectile()` flags enum | enum | Return/output flags: `_flyby_of_current_player`, `_projectile_hit`, `_projectile_hit_monster`, `_projectile_hit_floor`, `_projectile_hit_media`, `_projectile_hit_landscape`, `_projectile_hit_scenery` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ProjectileList` | `vector<projectile_data>` | global | Dynamic array of all active projectiles currently in the map |
| `MAXIMUM_PROJECTILES_PER_MAP` | macro | global | Resolves to `get_dynamic_limit(_dynamic_limit_projectiles)` at runtime; configurable per-map limit |

## Key Functions / Methods

### new_projectile
- Signature: `short new_projectile(world_point3d *origin, short polygon_index, world_point3d *_vector, angle delta_theta, short type, short owner_index, short owner_type, short intended_target_index, _fixed damage_scale)`
- Purpose: Create a new active projectile in the world
- Inputs: spawn location (3D point + polygon), velocity vector, rotation offset, projectile type, owner identity, intended target (for guided projectiles), damage multiplier
- Outputs/Return: projectile index (short) in `ProjectileList`
- Side effects: Appends to `ProjectileList`; may trigger sound/effects

### preflight_projectile
- Signature: `bool preflight_projectile(world_point3d *origin, short polygon_index, world_point3d *_vector, angle delta_theta, short type, short owner, short owner_type, short *target_index)`
- Purpose: Validate projectile creation before committing; check feasibility
- Inputs: Same as `new_projectile` (minus damage_scale)
- Outputs/Return: boolean (feasible or not)
- Side effects: None (read-only validation); may modify `*target_index` on output
- Notes: Used to test if `new_projectile()` would succeed

### translate_projectile
- Signature: `uint16 translate_projectile(short type, world_point3d *old_location, short old_polygon_index, world_point3d *new_location, short *new_polygon_index, short owner_index, short *obstruction_index, short *last_line_index, bool preflight)`
- Purpose: Move a projectile from old to new location; detect and report collisions
- Inputs: projectile type, old/new positions and polygon indices, owner index, preflight flag
- Outputs/Return: `uint16` flags (hit monster/floor/media/landscape/scenery, or flyby); output parameters: `*new_polygon_index`, `*obstruction_index`, `*last_line_index`
- Side effects: None (collision detection only; actual damage/detonation handled by caller)
- Notes: May hit different location than `new_location` (media boundary penetration case)

### move_projectiles
- Signature: `void move_projectiles(void)`
- Purpose: Update all active projectiles; assumes 1-tick time step
- Inputs: None (operates on global `ProjectileList`)
- Outputs/Return: None
- Side effects: Updates all projectile positions, handles contrails, detonates on collision, removes dead projectiles

### detonate_projectile
- Signature: `void detonate_projectile(world_point3d *origin, short polygon_index, short type, short owner_index, short owner_type, _fixed damage_scale)`
- Purpose: Trigger detonation/explosion at location (not tied to a specific projectile object)
- Inputs: detonation location, type (determines explosion effect), owner identity, damage scale
- Outputs/Return: None
- Side effects: Spawns explosion effect/particles, damages nearby entities

### remove_projectile / remove_all_projectiles
- Signature: `void remove_projectile(short projectile_index); void remove_all_projectiles(void)`
- Purpose: Delete one or all projectiles from the active list
- Inputs: projectile index (for single removal)
- Side effects: Frees slot in `ProjectileList`

### orphan_projectiles
- Signature: `void orphan_projectiles(short monster_index)`
- Purpose: Clear owner reference when the owner (monster/NPC) dies
- Inputs: monster index that was the owner
- Side effects: Updates `owner_index` to `NONE` for affected projectiles

### ProjectileIsGuided
- Signature: `bool ProjectileIsGuided(short Type)`
- Purpose: Query whether a projectile type is guided/homing
- Inputs: projectile type enum
- Outputs/Return: boolean

### Serialization Functions
- `unpack_projectile_data(uint8 *Stream, projectile_data *Objects, size_t Count)` / `pack_projectile_data(...)`  
  Pack/unpack runtime projectile state (for save/load)
- `unpack_projectile_definition(uint8 *Stream, size_t Count)` / `pack_projectile_definition(...)`  
  Pack/unpack projectile definitions (weapon parameters, not exposed to outside)
- `init_projectile_definitions()`  
  Initialize projectile parameters from resource/definition data

### Utility / Asset Loading
- `mark_projectile_collections(short type, bool loading)` ΓÇô Reference-count resource collections
- `load_projectile_sounds(short type)` ΓÇô Load audio assets for a projectile type
- `drop_the_ball(...)` ΓÇô Special-case physics object spawner
- `get_projectile_data(const short projectile_index)` ΓÇô Accessor to raw projectile_data struct

## Control Flow Notes
Projectiles are ticked once per frame:
1. **Creation**: `new_projectile()` or `detonate_projectile()` (for detonations without persistent object)
2. **Update**: `move_projectiles()` iterates all active projectiles, calls `translate_projectile()` to detect collisions, updates position/contrail state
3. **Cleanup**: On collision or age, projectile is removed via `remove_projectile()` or detonates
4. **Save/Load**: Serialization functions pack/unpack projectile state for persistence

Projectile lifecycle is independent of object lifecycle; a projectile may outlive its owner or exist without a persistent object.

## External Dependencies
- **dynamic_limits.h**: Provides `get_dynamic_limit()` to fetch runtime limit on max projectiles
- **world.h**: Defines `angle`, `world_distance`, `world_point3d`, `world_vector3d`, trigonometry & distance functions
- Implicit: engine defines `_fixed` (fixed-point numeric type), `NONE` constant, and polygon/monster/object indexing conventions
