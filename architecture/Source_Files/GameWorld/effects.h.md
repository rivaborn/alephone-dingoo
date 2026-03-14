# Source_Files/GameWorld/effects.h

## File Purpose
Header for the game engine's visual effects system. Defines the interface for creating, managing, and updating game effects (explosions, blood splatters, splashes, contrails, etc.) that occur during gameplay. Effects are temporary or persistent visual phenomena tied to world events.

## Core Responsibilities
- Define 70+ effect types (explosions, contrails, blood splashes, liquid splashes, detonations, etc.)
- Manage active effect data structure and list
- Provide creation/destruction interface for effects
- Handle per-tick effect updates
- Manage effect serialization/deserialization for save/load
- Track dynamic limits on active effects per map
- Provide teleport effect operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `effect_data` | struct | 32-byte effect state: type, object reference, flags, timing, special data |
| `effect_types` enum | enum | 75+ distinct effect type constants (e.g., `_effect_rocket_explosion`, `_effect_player_blood_splash`) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `EffectList` | `vector<effect_data>` | global | Dynamic array of all active effects in the world |
| `effects` | macro ΓåÆ `&EffectList[0]` | global | Backward-compatibility pointer to first effect (legacy interface) |

## Key Functions / Methods

### new_effect
- Signature: `short new_effect(world_point3d *origin, short polygon_index, short type, angle facing)`
- Purpose: Create a new effect at a world position.
- Inputs: World position, containing polygon, effect type, facing angle.
- Outputs/Return: Effect index (handle to the created effect).
- Side effects: Allocates and registers new effect in `EffectList`; increments active effect count.
- Calls: (Implementation in effects.c, not visible here.)
- Notes: Returns index for later reference; respects `MAXIMUM_EFFECTS_PER_MAP` limit.

### update_effects
- Signature: `void update_effects(void)`
- Purpose: Advance all active effects by one tick.
- Inputs: None (operates on global `EffectList`).
- Outputs/Return: None.
- Side effects: Modifies delay counters, may remove expired effects, updates animation/state.
- Calls: (Implementation in effects.c; likely calls per-effect update logic.)
- Notes: Called once per game tick. Comment says assumes ╬öt==1 tick.

### remove_effect
- Signature: `void remove_effect(short effect_index)`
- Purpose: Remove a specific active effect by index.
- Inputs: Effect index.
- Outputs/Return: None.
- Side effects: Frees effect slot; updates `EffectList`.
- Calls: (Implementation details in effects.c.)

### get_effect_data
- Signature: `effect_data *get_effect_data(const short effect_index)`
- Purpose: Retrieve pointer to effect data at index.
- Inputs: Effect index.
- Outputs/Return: Pointer to `effect_data` struct.
- Side effects: None.
- Notes: Accessor; marked const input.

### Serialization functions
- `unpack_effect_data`, `pack_effect_data`, `unpack_effect_definition`, `pack_effect_definition`
  - Purpose: Binary serialization for save/load. Stream-based, operate on byte buffers.
  - Inputs: Byte stream pointer, data object(s), count.
  - Outputs/Return: Updated stream pointer (consumed bytes).
  - Side effects: Writes to/reads from memory stream.

### Initialization & Utility
- `init_effect_definitions()` ΓÇô Initialize effect definitions from resource/XML.
- `remove_all_nonpersistent_effects()` ΓÇô Flush temporary effects.
- `mark_effect_collections(type, loading)` ΓÇô Mark resource collections for loading/unloading.
- `teleport_object_in/out(object_index)` ΓÇô Special teleport effect triggers.

## Control Flow Notes
- Integrated into main game loop: `update_effects()` called once per tick (~30 Hz).
- Effects created on demand via `new_effect()` when game events occur (weapon fire, damage, splashes, etc.).
- Respects dynamic limits settable from resource fork / XML; max effects per map is configurable at runtime.

## External Dependencies
- **Includes:** `dynamic_limits.h` (for `get_dynamic_limit()` macro call in `MAXIMUM_EFFECTS_PER_MAP`).
- **Types used:** `world_point3d`, `angle` (defined elsewhere, likely in geometry headers).
- **External symbols:** `get_dynamic_limit(_dynamic_limit_effects)` ΓÇô retrieves current effect limit.
- **Language features:** Uses C++ `vector<>` (modern C++ container, not pure C).
