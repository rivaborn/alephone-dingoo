# Source_Files/GameWorld/monsters.h

## File Purpose
Header file defining the monster/NPC system for the Marathon game engine (Aleph One). Declares structures, enums, and function interfaces for managing all non-player entitiesΓÇötheir types, behaviors, state transitions, damage, and lifecycle.

## Core Responsibilities
- Define monster data structures (`monster_data`) and persistent state (type, vitality, flags, action, mode)
- Enumerate ~40 monster types (ticks, cyborgs, hunters, civilians, yeties, defenders, etc.)
- Manage monster creation, despawning, and frame-by-frame updates
- Provide activation/deactivation and state queries via flag macros
- Handle damage application, death events, and collision/pathfinding integration
- Support sound-based and trigger-based monster activation systems
- Enable serialization (packing/unpacking) and XML-based configuration loading

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `monster_data` | struct (64 bytes) | Core entity: type, vitality, flags, action/mode, target index, velocities, path, activation bias, sound location |
| Monster type enum | enum | ~40 types: marines, ticks, compilers, fighters, cyborgs, hunters, yeties, mothers, etc. |
| Monster action enum | enum | 12 states: stationary, moving, attacking (close/far), being hit, dying (hard/soft/flaming), teleporting |
| Monster mode enum | enum | 5 targeting modes: locked, losing_lock, lost_lock, unlocked, running |
| Monster flags enum | enum | 6 persistent flags: promoted, demoted, never_activated, blind, deaf, teleport_on_deactivate |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MonsterList` | `vector<monster_data>` | global | All spawned monster entities |
| `MAXIMUM_MONSTERS_PER_MAP` | macro | global | Dynamic limit from resource configuration |
| `LOCAL_INTERSECTING_MONSTER_BUFFER_SIZE` | macro | global | Collision detection buffer (dynamic limit) |
| `GLOBAL_INTERSECTING_MONSTER_BUFFER_SIZE` | macro | global | Global collision buffer (dynamic limit) |

## Key Functions / Methods

### move_monsters
- Signature: `void move_monsters(void)`
- Purpose: Main NPC update loop; advances all active monsters each game tick
- Inputs: None (reads global `MonsterList`)
- Outputs/Return: None
- Side effects: Updates position, action state, timers, velocities; may change activation status
- Calls: (not inferable from header)
- Notes: Assumes 1-tick delta; primary per-frame update entry point

### new_monster
- Signature: `short new_monster(struct object_location *location, short monster_code)`
- Purpose: Spawn a new monster at a map location
- Inputs: spawn location, monster type code
- Outputs/Return: monster index; NONE on failure
- Side effects: Allocates slot in `MonsterList`; initializes monster data; marks collections for loading
- Calls: `mark_monster_collections()`
- Notes: Vitality may be deferred until first activation; location must be valid polygon

### damage_monster
- Signature: `void damage_monster(short monster_index, short aggressor_index, short aggressor_type, world_point3d *epicenter, struct damage_definition *damage, short projectile_index)`
- Purpose: Apply damage to a single monster
- Inputs: target index, aggressor identity, damage epicenter location, damage definition, projectile source
- Outputs/Return: None
- Side effects: Reduces vitality; triggers hit animation; sets `RECOVERING_FROM_HIT` flag; may invoke death
- Calls: `monster_died()` on fatal damage
- Notes: Aggressor tracking affects later target selection; damage type influences animation

### monster_died
- Signature: `void monster_died(short target_index)`
- Purpose: Handle death event; cleanup and state transitions
- Inputs: index of dead monster
- Outputs/Return: None
- Side effects: Updates target lists for other monsters; deactivates/removes from active list
- Calls: (not inferable from header)
- Notes: Triggered automatically when vitality Γëñ 0; may cascade to remove threat from allied monsters

### activate_nearby_monsters
- Signature: `void activate_nearby_monsters(short target_index, short caller_index, short flags)`
- Purpose: Activate dormant monsters in response to sound or trigger events
- Inputs: epicenter target, activation caller origin, flags (zone crossing, sound trigger, activation bias, etc.)
- Outputs/Return: None
- Side effects: Marks inactive monsters as active; updates activation bias; triggers sound loading
- Calls: `activate_monster()`, `load_monster_sounds()`
- Notes: Respects editor-set activation biases; can activate deaf/invisible monsters if `_activate_deaf_monsters` or `_activate_invisible_monsters` flagged

## Control Flow Notes
- **Initialization:** `initialize_monsters()` system setup; `initialize_monsters_for_new_level()` loads level monsters from map data
- **Per-frame update:** `move_monsters()` called each tick to advance all NPC logic
- **Damage/events:** `damage_monster()` and `monster_died()` triggered by collision or effect radius
- **Activation:** Monsters inactive by default; triggered by sound proximity, explicit activation, or trigger flags
- **Serialization:** Pack/unpack functions support persistent level state

## External Dependencies
- **Includes:** `dynamic_limits.h` (runtime limits), `XML_ElementParser.h` (config parsing), `<vector>` (STL storage)
- **Defined elsewhere:** `world_point3d`, `world_distance`, `angle`, `object_location`, `damage_definition`, `monster_definition`, `object_data`
