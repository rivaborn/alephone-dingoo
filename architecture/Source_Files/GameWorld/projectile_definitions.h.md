# Source_Files/GameWorld/projectile_definitions.h

## File Purpose

Defines the static properties and behavior flags for all projectile types in the game engine. Contains a comprehensive data-driven lookup table of ~40 projectile definitions initialized at engine startup, each specifying damage, visual effects, physics behavior, and behavioral flags.

## Core Responsibilities

- Define projectile behavior flags (e.g., guided, gravity-affected, penetrating, melee-capable)
- Declare the `projectile_definition` struct containing all per-projectile properties
- Initialize `original_projectile_definitions[]` constant array with full definitions for all ~40 projectile types (player weapons, alien projectiles, special effects)
- Maintain a mutable `projectile_definitions[]` array for runtime modifications
- Provide serialization/deserialization functions for save/load and network transmission

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| (unnamed) | enum | Projectile flags: `_guided`, `_affected_by_gravity`, `_penetrates_media`, `_melee_projectile`, `_becomes_item_on_detonation`, etc. Control behavior modifiers. |
| `projectile_definition` | struct | Aggregates all properties for a single projectile: collection/shape ID, effects, damage profile, speed, range, sound pitch, detonation sound, flags. |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `projectile_definitions` | `struct projectile_definition[]` | static | Runtime-mutable array of projectile definitions; can be patched at runtime. |
| `original_projectile_definitions` | `const struct projectile_definition[]` | static | Immutable reference array initialized with default projectile properties. ~40 entries (rockets, grenades, bullets, alien bolts, special projectiles). |

## Key Functions / Methods

### unpack_projectile_definition
- **Signature:** `uint8 *unpack_projectile_definition(uint8 *Stream, projectile_definition *Objects, size_t Count)`
- **Purpose:** Deserialize a stream of bytes into an array of `projectile_definition` structures.
- **Inputs:** Byte stream, target object array, count of objects to unpack.
- **Outputs/Return:** Pointer to the byte position after unpacking (for chained reads).
- **Side effects (global state, I/O, alloc):** Populates the provided `Objects` array; does not allocate.
- **Notes:** Declartion only; implementation elsewhere. Used for loading save files or network data.

### pack_projectile_definition
- **Signature:** `uint8 *pack_projectile_definition(uint8 *Stream, projectile_definition *Objects, size_t Count)`
- **Purpose:** Serialize an array of `projectile_definition` structures into a byte stream.
- **Inputs:** Target byte stream, source object array, count of objects to pack.
- **Outputs/Return:** Pointer to the byte position after packing.
- **Side effects (global state, I/O, alloc):** Writes to the provided stream; does not allocate.
- **Notes:** Declaration only; implementation elsewhere. Used for saving state or network transmission.

## Control Flow Notes

This file is a **data-driven configuration header** with no executable control flow. It is included at compile time to statically initialize the `original_projectile_definitions[]` array. The engine likely loads or copies these definitions during initialization (`startup/init` phase) to set up the global `projectile_definitions[]` array. Runtime code that spawns or simulates projectiles indexes into these arrays to look up behavior, damage, and effects. The `pack`/`unpack` functions integrate with save/load and multiplayer serialization systems.

## External Dependencies

- **Types referenced (defined elsewhere):** `world_distance`, `_fixed`, `damage_definition`, `int16`, `uint32`, `uint8`, `size_t`
- **Macros:** `WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_THREE_FOURTHS`, `WORLD_ONE/N` (distance scaling), `BUILD_COLLECTION()`, `NONE`
- **External symbols (enum values):** Damage types (`_damage_explosion`, `_damage_projectile`, `_damage_flame`, etc.), effect IDs (`_effect_rocket_explosion`, `_effect_grenade_explosion`, etc.), collection IDs (`_collection_rocket`, `_collection_fighter`, `_collection_compiler`, etc.), sound IDs (`_snd_rocket_flyby`, `_snd_fusion_flyby`, etc.), flags (`_alien_damage`)
- **Guard:** `#ifndef DONT_REPEAT_DEFINITIONS` allows conditional inclusion of the definitions array.
