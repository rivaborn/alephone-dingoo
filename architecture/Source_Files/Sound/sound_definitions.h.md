# Source_Files/Sound/sound_definitions.h

## File Purpose
Defines core data structures, enumerations, and constants for the game's sound management system. Declares static arrays for sound behavior profiles, ambient sound indices, and random sound indices. Originally contained static sound definitions; now supports dynamic allocation.

## Core Responsibilities
- Define sound behavior categories (quiet, normal, loud) and their depth-based volume falloff curves
- Specify bit flags controlling sound behavior (restart restrictions, pitch changes, obstruction handling, ambient flag)
- Define probability/chance constants for conditional sound playback
- Declare core structures for sound definitions, file headers, depth curves, and behaviors
- Initialize static lookup tables for ambient and random sound mappings
- Track active sound definition count via global counter

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `sound_definition` | struct | Core sound entity; holds code, behavior index, flags, probability, pitch range, permutation metadata, playback offsets, last-played timestamp, and loaded sound pointer/size |
| `sound_file_header` | struct | Describes sound data file layout; version, tag, source count, sound count, followed by `source_count ├ù sound_count` sound definitions |
| `sound_behavior_definition` | struct | Pairs obstructed and unobstructed depth curves; defines volume falloff behavior for a sound category |
| `depth_curve_definition` | struct | Specifies maximum and minimum volume levels and their associated distances; governs spatial audio attenuation |
| `ambient_sound_definition` | struct | Wraps a sound index for ambient looping sounds |
| `random_sound_definition` | struct | Wraps a sound index for random event sounds |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sound_behavior_definitions[]` | `sound_behavior_definition[3]` | static | Lookup table for quiet, normal, and loud sound behavior curves; indexed by behavior enum |
| `ambient_sound_definitions[]` | `ambient_sound_definition[]` | static | Maps ambient sound indices (water, wind, machinery, alien, etc.) |
| `random_sound_definitions[]` | `random_sound_definition[]` | static | Maps random ambient sound indices (water drip, explosions, etc.) |
| `number_of_sound_definitions` | `int32` | static | Counter tracking how many sound definitions are currently in use (supports dynamic allocation) |

## Key Functions / Methods
None. This is a pure data definition header.

## Control Flow Notes
Not inferable. This file is purely declarative; it provides constants and data structures used by sound management code elsewhere (presumably in corresponding `.c` files). It does not directly participate in engine initialization, frame updates, or shutdown cycles.

## External Dependencies
- Macros: `FOUR_CHARS_TO_INT()` (converts four ASCII chars to an int; used for file tag validation)
- Types: `_fixed` (fixed-point number), `int16/int32`, `uint16/uint32`, `uint8` (platform-specific typedefs)
- Constants: `MAXIMUM_SOUND_VOLUME`, `WORLD_ONE`, `MAXIMUM_PERMUTATIONS_PER_SOUND` (defined elsewhere)
- Enumerated sound codes: `_snd_water`, `_snd_teleport_in`, `_snd_magnum_firing`, etc. (defined elsewhere, likely in an enumeration header)
- Count constants: `NUMBER_OF_SOUND_BEHAVIOR_DEFINITIONS`, `NUMBER_OF_AMBIENT_SOUND_DEFINITIONS`, `NUMBER_OF_RANDOM_SOUND_DEFINITIONS` (defined elsewhere)

**Notes:**
- File format is `'snd2'` tag to maintain Marathon Infinity compatibility.
- Sound definitions originally used static allocation (large commented-out block); converted to dynamic allocation to support runtime sound definition management.
- Probability thresholds use `AbsRandom()` comparisons with FIXED-point fractional percentages (e.g., `_fifty_percent = 32768*5/10`).
- Sounds support up to 5 permutations per definition for audio variety.
- `_fixed` pitch range `[low_pitch, high_pitch]` allows runtime pitch variation; zero values have special meaning (0 ΓåÆ use default FIXED_ONE; high_pitch=0 ΓåÆ use low_pitch).
