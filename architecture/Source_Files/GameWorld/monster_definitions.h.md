# Source_Files/GameWorld/monster_definitions.h

## File Purpose

Defines all monster types and their combat/behavioral properties in the Marathon/Aleph One game engine. Contains complete specifications for 30+ monster variants, including statistics (health, speed, attack patterns), class relationships (friend/foe), and rendering/sound data. Serves as the central configuration for all AI-controlled creatures.

## Core Responsibilities

- Define monster class hierarchy and allegiance relationships (Pfhor aliens, Compilers, Defenders, Yetis, natives, player, humans)
- Specify combat parameters: vitality, attack types/ranges, projectile capabilities, speeds
- Configure visual representation: shape collections, animation frames, teleport effects, size variants (major/minor/tiny)
- Set sensory parameters: visual range, intelligence levels, sound effects
- Control environmental interactions: door-opening capability, ledge navigation, media-specific behaviors (lava/water/goo fears)
- Provide pack/unpack serialization for monster definitions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `attack_definition` | struct | Specifies an attack's projectile type, range, repetitions, firing offset (dx/dy/dz), accuracy/error |
| `monster_definition` | struct | Complete 128+ byte configuration: vitality, immunities, flags, class/faction, audio, geometry, AI params, dual attacks (melee/ranged) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `monster_definitions` | `monster_definition[NUMBER_OF_MONSTER_TYPES]` | global | Runtime-modifiable array of active monster definitions |
| `original_monster_definitions` | `const monster_definition[NUMBER_OF_MONSTER_TYPES]` | static | Hardcoded defaults for ~30 monster types (marine, ticks, compilers, fighters, juggernauts, yetis, defenders, VacBobs) |

## Key Functions / Methods

### unpack_monster_definition
- Signature: `uint8 *unpack_monster_definition(uint8 *Stream, monster_definition *Objects, size_t Count)`
- Purpose: Deserialize monster definitions from a binary data stream (e.g., level data)
- Inputs: Byte stream pointer, destination array, count of definitions to unpack
- Outputs/Return: Updated stream pointer (position after last unpacked byte)
- Side effects: Modifies target `Objects` array in-place
- Calls: Not inferable from this file
- Notes: Paired with `pack_monster_definition` for save/load

### pack_monster_definition
- Signature: `uint8 *pack_monster_definition(uint8 *Stream, monster_definition *Objects, size_t Count)`
- Purpose: Serialize monster definitions to a binary stream for storage/transmission
- Inputs: Target stream, source array, count
- Outputs/Return: Updated stream pointer
- Side effects: Writes to stream; modifies no other state
- Calls: Not inferable from this file
- Notes: Inverse of unpack; enables persistent game state

## Control Flow Notes

This is a **data definition file** with no active control flow. Loading occurs once at engine startup via init functions (inferred): `original_monster_definitions` is copied into the mutable `monster_definitions` array, which is then indexed by monster-spawn code. The pack/unpack functions are invoked during level load/save cycles. Monster instances are created dynamically; this file defines the template/spec for each type.

## External Dependencies

- **effects.h**: Effect type constants referenced in monster definitions (e.g., `_effect_fighter_blood_splash`, `_effect_juggernaut_spark`)
- **projectiles.h**: Projectile type constants for melee/ranged attacks (e.g., `_projectile_staff`, `_projectile_compiler_bolt_major`)
- **Macro constants** (defined elsewhere): `WORLD_ONE`, `FIXED_ONE`, `TICKS_PER_SECOND`, `NUMBER_OF_ANGLES`, `BUILD_COLLECTION`, `FLAG`, `NONE`, `UNONE`, `QUARTER_CIRCLE`
- **Type definitions** (from broader codebase): `int16`, `uint32`, `_fixed`, `angle`, `world_distance`, `shape_descriptor`, `damage_definition`

---

## Detailed Notes

**Monster Class System:** Uses bitfield enums (e.g., `_class_player_bit`, `_class_fighter_bit`) to define allegiance groups. Macros like `TYPE_IS_ENEMY` check class compatibility.

**Attack Model:** Each monster has separate melee and ranged `attack_definition` entries. Melee is proximity-based; ranged fires projectiles at distance. `attack_frequency` (in ticks) controls how often attacks occur.

**Behavior Flags:** 28+ feature flags control special behaviors (invisibility, berserker rampages, kamikazes, teleportation under media, etc.).

**Speed Enums:** Predefined constants (`_speed_slow` through `_speed_insane`) define movement and projectile velocities as `WORLD_ONE / divisor`.

**Size Variants:** Monsters have major (stronger), minor (weaker), and tiny versions. Properties scale (vitality, visual arc, speed adjustments).
