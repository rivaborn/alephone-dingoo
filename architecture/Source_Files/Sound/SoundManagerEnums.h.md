# Source_Files/Sound/SoundManagerEnums.h

## File Purpose
Header file containing enumeration constants for the Aleph One sound manager system. Defines sound type identifiers, audio configuration flags, and sound propagation properties used throughout the game engine.

## Core Responsibilities
- Define ambient sound codes (water, machinery, wind, etc.) and their enumerated IDs
- Define random sound codes (drips, explosions) with their enumerated IDs
- Define primary sound effect codes for weapons, entities, UI, and environmental events
- Define sound volume configuration constants and bit widths
- Define audio source format types (8-bit/16-bit, 22kHz sample rates)
- Define sound manager initialization flags (stereo, Doppler, ambient tracking, memory options)
- Define sound obstruction and media interaction flags
- Define frequency adjustment constants for pitch modulation

## Key Types / Data Structures
NoneΓÇöthis file contains only unnamed enumerations defining constant values, not struct/class definitions.

## Global / File-Static State
None.

## Key Functions / Methods
NoneΓÇöthis is a pure definition file with no executable code.

## Control Flow Notes
Not applicableΓÇöthis file is purely declarative and consumed as constants by the SoundManager implementation and other audio-related modules.

## External Dependencies
- **FIXED_ONE** macro (defined elsewhere) ΓÇö used in frequency constants for fixed-point arithmetic (`_lower_frequency`, `_normal_frequency`, `_higher_frequency`)
- Consumed by: SoundManager.h/SoundManager.cpp and audio subsystem callers

## Notes
- Comments indicate some LP (Loren Petrich) additions and changes for extensibility beyond the original Marathon 2 sound set
- Several enums include a `NUMBER_OF_*` sentinel value for iteration/sizing
- Initialization flags have dual purpose: some are saved in prefs and constrain the sound system at runtime (marked `[prefs]`)
- Sound codes span three categories: ambient (looping background), random (occasional events), and main effects (triggered sounds)
