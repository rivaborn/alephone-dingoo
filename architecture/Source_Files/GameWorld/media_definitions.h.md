# Source_Files/GameWorld/media_definitions.h

## File Purpose
Defines static configuration data for all in-game media types (water, lava, goo, sewage, Jjaro). Specifies visual representation, damage properties, sound effects, and detonation effects for each medium the player can interact with.

## Core Responsibilities
- Define the `media_definition` struct schema (visual, audio, damage, effects properties)
- Initialize a global static array with configurations for 5 distinct media types
- Specify splash/detonation effects for small/medium/large interactions with each medium
- Configure ambient and interaction sound effects for each medium
- Define damage rules (frequency and type) when submerged in each medium
- Map each medium to its graphical representation (collection/shape)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `media_definition` | struct | Holds all properties for a single media type: visuals, damage, sounds, effects, transfer mode |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `media_definitions` | `media_definition[NUMBER_OF_MEDIA_TYPES]` | static | Initialized array of 5 media configurations (water, lava, goo, sewage, Jjaro); lookup table used during gameplay |

## Key Functions / Methods
None. This file is purely declarative data.

## Control Flow Notes
Initialization-time data. The `media_definitions` array is loaded once at startup and referenced during:
- **Rendering**: `collection`, `shape`, `transfer_mode` determine how to draw the medium
- **Physics/Collision**: `damage_frequency` and `damage` apply when entity is submerged
- **Audio**: `sounds` array provides ambient and action-triggered SFX
- **Particles/Effects**: `detonation_effects` and `submerged_fade_effect` trigger visual effects

## External Dependencies
- External enums/constants: `_media_water`, `_collection_walls1ΓÇô5`, `_xfer_normal`, `_effect_*` (splash/emergence), `_snd_*` (sound IDs), `_damage_lava`, `_damage_goo`, etc.
- External struct: `damage_definition`
- Constants: `NONE`, `FIXED_ONE`, `NUMBER_OF_MEDIA_TYPES`, `NUMBER_OF_MEDIA_DETONATION_TYPES`, `NUMBER_OF_MEDIA_SOUNDS`

All symbols defined elsewhere. Not inferable from this file.
