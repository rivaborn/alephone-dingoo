# Source_Files/GameWorld/effects.cpp

## File Purpose
Implements the runtime effect system for the game engine. Effects are temporary visual and audio phenomena (explosions, splashes, contrails, teleportation) spawned at world locations and animated until completion. This file manages effect instance creation, per-frame updates, removal, and persistence state tracking.

## Core Responsibilities
- Create and track active effect instances at world locations
- Update effect animations and detect termination conditions
- Manage visibility and sound playback for delayed/invisible effects
- Handle specialized teleportation effects that coordinate with game objects
- Serialize/deserialize effect data for save/load
- Mark required shape collections for resource loading

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `effect_data` | struct | Runtime instance of an active effect; tracks type, linked object index, flags, animation delay, and custom data (32 bytes) |
| `effect_definition` | struct | Configuration for an effect type (collection, shape, sound pitch, behavioral flags, delay timing) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `effects` | `vector<effect_data>` | extern (defined in map.cpp) | Pool of active effects, sized to `MAXIMUM_EFFECTS_PER_MAP` |
| `effect_definitions` | `effect_definition[]` | extern (from effect_definitions.h) | Metadata for each effect type; initialized at startup |
| `get_effect_definition` | function pointer | static | Accessor; wrapped in bounds-checking for idiot-proofing |

## Key Functions / Methods

### new_effect
- **Signature:** `short new_effect(world_point3d *origin, short polygon_index, short type, angle facing)`
- **Purpose:** Spawn a new effect instance at a location. If sound-only, play sound and return. Otherwise allocate effect slot, create map object, and initialize effect data.
- **Inputs:** Origin location, polygon context, effect type enum, facing angle
- **Outputs/Return:** Effect index (`NONE` on failure or invalid polygon)
- **Side effects:** Allocates effect slot, creates map object, plays sounds, may load collections
- **Calls:** `get_effect_definition`, `get_shape_animation_data`, `play_world_sound`, `new_map_object3d`, `get_object_data`, `SET_OBJECT_OWNER`, `SET_OBJECT_INVISIBILITY`
- **Notes:** Delay-enabled effects spawn invisible; sound-only effects skip object creation; media effects are flagged on the object

### update_effects
- **Signature:** `void update_effects(void)`
- **Purpose:** Per-frame update loop; decrement delays, animate objects, detect animation completion, remove expired effects, make twins visible.
- **Inputs:** None (operates on global effects array and linked objects)
- **Outputs/Return:** None
- **Side effects:** Modifies effect delay counters, object visibility, calls animate_object and remove_effect
- **Calls:** `get_object_data`, `get_effect_definition`, `animate_object`, `remove_effect`, `play_object_sound`, `SET_OBJECT_INVISIBILITY`
- **Notes:** Checks both animation-loop and transfer-mode-finished flags for termination; conditional twin visibility for teleport effects

### remove_effect
- **Signature:** `void remove_effect(short effect_index)`
- **Purpose:** Deallocate effect and its associated map object.
- **Inputs:** Effect index
- **Outputs/Return:** None
- **Side effects:** Removes object from world, marks effect slot free
- **Calls:** `get_effect_data`, `remove_map_object`, `MARK_SLOT_AS_FREE`
- **Notes:** Assumes index is valid; no error checking

### remove_all_nonpersistent_effects
- **Signature:** `void remove_all_nonpersistent_effects(void)`
- **Purpose:** Cleanup routine; remove all effects flagged for termination on animation/transfer-mode loop.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls remove_effect on matching effects
- **Calls:** `get_effect_definition`, `remove_effect`
- **Notes:** Idiot-proofs by skipping effects with invalid definitions

### teleport_object_out / teleport_object_in
- **Signature:** `void teleport_object_out(short object_index)` / `void teleport_object_in(short object_index)`
- **Purpose:** Spawn coordinated teleportation effects. Out: create fold-out effect and hide object. In: create fold-in effect linked to destination object.
- **Inputs:** Object index to teleport
- **Outputs/Return:** None
- **Side effects:** Creates effects, modifies object visibility and transfer mode, plays teleport sound
- **Calls:** `get_object_data`, `new_effect`, `get_effect_data`, `play_object_sound`, `SET_OBJECT_INVISIBILITY`
- **Notes:** In-effect checks if object already has a teleport-in pending; effect.data holds object_index for twin visibility linkage; handles scale flags (enlarged/tiny)

### Packing / Unpacking Functions
- **Signatures:** `uint8 *unpack/pack_effect_data(uint8 *Stream, effect_data *Objects, size_t Count)` and `_effect_definition` variants
- **Purpose:** Serialize/deserialize effect state for save/load; advance stream pointer and assert correct byte count
- **Side effects:** Stream pointer advanced; assertions on size match
- **Notes:** Pack/unpack advance stream by 32 bytes per effect, 14 bytes per definition; skip 22 bytes of padding in effect_data

### mark_effect_collections
- **Signature:** `void mark_effect_collections(short effect_type, bool loading)`
- **Purpose:** Mark shape collections for load/unload based on effect type.
- **Inputs:** Effect type, loading flag
- **Outputs/Return:** None
- **Side effects:** Calls mark_collection_for_loading/unloading
- **Calls:** `get_effect_definition`, `mark_collection_for_loading`, `mark_collection_for_unloading`
- **Notes:** Idiot-proofs by skipping invalid definitions

### get_effect_data / get_effect_definition
- **Signature:** `effect_data *get_effect_data(short effect_index)` / `effect_definition *get_effect_definition(short type)`
- **Purpose:** Bounds-checked accessors with assertions on validity.
- **Inputs:** Index/type
- **Outputs/Return:** Pointer to struct or implicit fatal assert
- **Notes:** Validate slot-in-use for effect_data; no explicit bounds on definition (will fault if out of range)

## Control Flow Notes
Effects are passive game-world entities. Creation is ad-hoc (projectile detonation, splash on collision, etc.). Each frame, `update_effects` animates all active effects and removes those whose animations terminate. Teleport effects are special: they coordinate object visibility and audio across a pair of effects (outΓåÆin), using the effect.data field to link source object. All effects are cleaned on level unload via `remove_all_nonpersistent_effects`.

## External Dependencies
- **map.h** ΓÇö world geometry, object management, animation macros, slot-in-use macros
- **interface.h** ΓÇö shape animation metadata (`get_shape_animation_data`)
- **SoundManager.h** ΓÇö `play_world_sound`, `play_object_sound`
- **Packing.h** ΓÇö stream serialization utilities
- **effect_definitions.h** ΓÇö effect type enums, flags, default definition array
