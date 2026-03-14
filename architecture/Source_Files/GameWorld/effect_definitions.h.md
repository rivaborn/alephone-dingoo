# Source_Files/GameWorld/effect_definitions.h

## File Purpose
Defines all visual and audio effects for the game world, including weapon detonations, blood splashes, media interactions, and environmental effects. Provides configuration data structures and default effect definitions, plus serialization utilities for save/load.

## Core Responsibilities
- Declares `effect_definition` struct to configure animation source, audio pitch, and timing behavior for each effect type
- Defines flags controlling effect termination conditions, audio-only modes, and media interaction
- Initializes `original_effect_definitions[]` array mapping ~60+ effect enum IDs to animation/sound configurations
- Manages runtime copy `effect_definitions[]` for potential modifications
- Exports pack/unpack functions for serializing effect data to/from streams

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `effect_definition` | struct | Holds collection ID, shape ID, sound pitch, flags, and delay values for one effect |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `effect_definitions` | `struct effect_definition[]` | static | Runtime effect array; likely initialized from `original_effect_definitions` |
| `original_effect_definitions` | `const struct effect_definition[]` | global | Default/immutable effect configurations indexed by effect type enum |

## Key Functions / Methods

### unpack_effect_definition
- Signature: `uint8 *unpack_effect_definition(uint8 *Stream, effect_definition *Objects, size_t Count)`
- Purpose: Deserialize effect definitions from a byte stream (e.g., loaded data)
- Inputs: byte stream pointer, destination effect array, count of objects to unpack
- Outputs/Return: updated stream pointer (position after unpacked data)
- Side effects: writes to `Objects` array
- Calls: Not visible (defined elsewhere, likely in .c file)
- Notes: Used by 1-2-3 converter for save/load

### pack_effect_definition
- Signature: `uint8 *pack_effect_definition(uint8 *Stream, effect_definition *Objects, size_t Count)`
- Purpose: Serialize effect definitions to a byte stream (e.g., for saving)
- Inputs: destination byte stream, effect array source, count of objects to pack
- Outputs/Return: updated stream pointer
- Side effects: writes to stream
- Calls: Not visible (defined elsewhere)
- Notes: Used by 1-2-3 converter for save/load

## Control Flow Notes
This is purely declarative configuration dataΓÇöno control flow. The effect system is likely driven by a separate effects engine that:
1. Looks up effect IDs to retrieve `effect_definitions[id]`
2. Plays associated animation (collection/shape) and sound (pitch, delay)
3. Terminates based on flags (`_end_when_animation_loops`, `_end_when_transfer_animation_loops`, `_sound_only`)

## External Dependencies
- **Collection constants** (e.g., `_collection_rocket`, `_collection_fighter`, `_collection_cyborg`): defined elsewhere, identify animation sprite sheets
- **Frequency constants** (e.g., `_normal_frequency`, `_higher_frequency`, `_lower_frequency`): pitch multipliers for sound playback
- **Sound constants** (e.g., `_snd_teleport_in`, `NONE`): sound IDs to play with effect
- **Macro** `BUILD_COLLECTION()`: packs collection variant; likely used for alternate art assets
- **Constants** `TICKS_PER_SECOND`, `NUMBER_OF_EFFECT_TYPES`: game timing/bounds

**Note**: Header guard typo: `__EFFECT_DEFINTIIONS_H` should be `__EFFECT_DEFINITIONS_H` (misspelled "DEFINITIONS").
