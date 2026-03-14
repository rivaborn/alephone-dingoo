# Source_Files/GameWorld/platform_definitions.h

## File Purpose
Defines platform type configurations for the Aleph One game engine. Contains a lookup table of platform definitions (doors, platforms, etc.) with their associated sounds, flags, and damage properties. This is a data-driven initialization file used to configure all platform behaviors in the game world.

## Core Responsibilities
- Define sound event enum for platform audio callbacks
- Specify the `platform_definition` structure that aggregates all properties for a platform type
- Initialize a global array of platform definitions for all platform types (8 types: SPHT doors, split doors, locked doors, platforms, heavy doors, Pfhor doors/platforms, etc.)
- Associate each platform type with its default configuration flags, damage model, and audio assets

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `platform_definition` | struct | Aggregates sounds (extension/contraction), key item index, static defaults, and damage definition for a single platform type |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `platform_definitions` | `platform_definition[]` | global | Static lookup table initialized with 8 platform type definitions; indexed by platform type enum |

## Key Functions / Methods
None. This is a data definition file containing only struct initialization.

## Notes
- Sound indices are referenced by enum (e.g., `_snd_spht_door_opening`, `_ambient_snd_spht_door`) but not defined here.
- Platform behavior flags (e.g., `_platform_is_player_controllable`, `_platform_extends_floor_to_ceiling`) are wrapped in a `FLAG()` macro.
- Damage model is specified as `(type, ?, min_damage, max_damage, damage_scale)` tuples.
- Several platform types use `NONE` for unused sound slots (e.g., basic platforms have no stopping/obstructed sounds).
- The array index order must match the platform type enum definition (not shown here).

## Control Flow Notes
Initialization only. This file is included once at engine startup to populate the platform database. Not part of frame/update/render loops. Configuration is read at world load time or platform creation time.

## External Dependencies
- **Undefined enums**: Sound IDs (`_snd_spht_door_opening`, `_ambient_snd_spht_door`, etc.), platform flags (`_platform_is_spht_door`, `_platform_extends_floor_to_ceiling`, etc.), platform speeds (`_slow_platform`, `_fast_platform`), delay types (`_long_delay_platform`, `_very_long_delay_platform`), damage types (`_damage_crushing`)
- **Undefined types**: `static_platform_data`, `damage_definition`
- **Undefined macros**: `FLAG()`, `NONE`, `FIXED_ONE`, `NUMBER_OF_PLATFORM_TYPES`
- **Header guard**: `__PLATFORM_DEFINITIONS_H`
