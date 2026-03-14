# Source_Files/Sound/song_definitions.h

## File Purpose
Header file defining the data structures and constants for song/music definitions in the game engine. Specifies how songs are structured with introduction, chorus, and trailer segments. Provides a table of song metadata for the game's audio system.

## Core Responsibilities
- Define the `sound_snippet` structure (offset-based audio segment representation)
- Define the `song_definition` structure (song metadata with playback control)
- Define song behavior flags (`_song_automatically_loops`)
- Provide macro for encoding random chorus counts (`RANDOM_COUNT`)
- Declare a global array of song definitions for engine initialization

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `sound_snippet` | struct | Represents a contiguous audio segment with start and end offsets |
| `song_definition` | struct | Complete song metadata including flag, segments, loop behavior, and timing |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `songs[]` | `struct song_definition[]` | global | Array of song definitions for all playable songs in the game |
| `RANDOM_COUNT(x)` | macro | global | Encodes random chorus counts by negating the count value |

## Key Types / Data Structures Details

### `sound_snippet`
- **Purpose**: Represents a segment of audio (intro, chorus, trailer) as byte offsets into a sound resource.
- **Fields**:
  - `int32 start_offset`: byte offset where segment begins
  - `int32 end_offset`: byte offset where segment ends

### `song_definition`
- **Purpose**: Complete metadata for a playable song, controlling structure and playback behavior.
- **Fields**:
  - `int16 flags`: bit flags (currently supports `_song_automatically_loops`)
  - `int32 sound_start`: offset into sound resource where this song begins
  - `struct sound_snippet introduction`: intro segment
  - `struct sound_snippet chorus`: chorus segment (repeatable)
  - `int16 chorus_count`: number of times to play chorus; negative = random count (use `RANDOM_COUNT()`)
  - `struct sound_snippet trailer`: ending segment
  - `int32 restart_delay`: ticks to wait before looping (scaled by `MACHINE_TICKS_PER_SECOND`)

## Control Flow Notes
This is a data definition header with no executable code. The `songs[]` array is likely populated at compile-time and referenced by audio playback systems during game init/runtime to construct and play songs. The structure supports dynamic chorus repetition (random or fixed) and looping behavior.

## External Dependencies
- `MACHINE_TICKS_PER_SECOND`: macro constant (defined elsewhere, platform-specific timing)
- Standard C integer types: `int16`, `int32` (from platform headers)

## Notes
- The `RANDOM_COUNT(x)` macro uses negation to encode randomness: negative counts trigger random selection, positive counts are literal.
- Only one song is defined in the `songs[]` array with all-zero segments (placeholder/template).
- Song data is offset-based (byte counts into a sound resource), suggesting songs are stored contiguously in binary sound files.
