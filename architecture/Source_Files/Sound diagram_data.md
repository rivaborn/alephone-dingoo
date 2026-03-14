# Source_Files/Sound/BasicIFFDecoder.cpp
## File Purpose
Implements the `BasicIFFDecoder` class, which parses and decodes uncompressed AIFF and WAV audio files. Handles format detection, header parsing, and provides sequential access to audio frame data with playback position tracking.

## Core Responsibilities
- Detect and validate AIFF and WAV file format headers
- Extract audio metadata (sample rate, channels, bit depth, endianness)
- Locate and track audio data chunk boundaries
- Provide buffered reading of audio frames
- Support seeking and rewinding within audio data
- Manage file lifecycle (open, read, close)

## External Dependencies
- **SDL:** `SDL_endian.h` ΓÇö endian-aware read macros (`SDL_ReadBE32`, `SDL_ReadLE32`, `SDL_ReadBE16`, `SDL_ReadLE16`) and RWops file I/O (`SDL_RWops`, `SDL_RWseek`, `SDL_RWtell`).
- **Decoder** (base class) ΓÇö defined elsewhere; provides virtual interface.
- **FileSpecifier, OpenedFile** ΓÇö file abstraction types defined elsewhere.
- **Unused includes:** `<vector>` and `AStream.h` are included but not referenced in this file.

# Source_Files/Sound/BasicIFFDecoder.h
## File Purpose
Declares `BasicIFFDecoder`, a concrete decoder for uncompressed AIFF and WAV audio formats. Provides file opening, streaming decode operations, and metadata queries for audio playback in the engine.

## Core Responsibilities
- Opens and parses AIFF/WAV file headers
- Decodes audio frames into client-supplied buffers
- Tracks playback state (position, completion)
- Provides audio format metadata (bit depth, channels, sample rate, endianness)
- Manages file lifecycle and seek operations

## External Dependencies
- **Includes**: `Decoder.h` (base class `StreamDecoder` / `Decoder`)
- **Types used but not defined**: `FileSpecifier`, `OpenedFile` (file I/O abstractions, defined elsewhere)
- **Implicit dependencies**: Audio file I/O, header parsing logic (not visible in header)

# Source_Files/Sound/Decoder.cpp
## File Purpose
Implements factory methods for instantiating audio decoders. Attempts to open a file with multiple decoder implementations (based on compilation flags) and returns the first one that succeeds, or null if all fail.

## Core Responsibilities
- Provide static factory method `StreamDecoder::Get()` to create stream decoders for compressed/streaming audio formats
- Provide static factory method `Decoder::Get()` to create decoders for uncompressed/complete-file audio formats
- Try multiple decoder implementations in priority order (SndfileDecoder ΓåÆ BasicIFFDecoder ΓåÆ VorbisDecoder ΓåÆ MADDecoder)
- Manage decoder lifetime using `auto_ptr` to avoid leaks during failed attempts

## External Dependencies
- `Decoder.h` ΓÇö base classes (StreamDecoder, Decoder)
- `BasicIFFDecoder.h`, `MADDecoder.h`, `SndfileDecoder.h`, `VorbisDecoder.h` ΓÇö concrete decoder implementations
- `<memory>` ΓÇö `std::auto_ptr`
- FileSpecifier (from FileHandler.h) ΓÇö file reference type

# Source_Files/Sound/Decoder.h
## File Purpose
Defines abstract interfaces for audio decoding of music and external sounds. Provides two decoder classes: `StreamDecoder` for streaming arbitrary audio data, and `Decoder` (extended with frame counting) for complete file decoders. Uses factory methods to instantiate appropriate concrete implementations based on file type.

## Core Responsibilities
- Define `StreamDecoder` abstract base class for streaming audio frame-by-frame
- Define `Decoder` abstract class extending `StreamDecoder` with total frame info
- Abstract audio format properties (bit depth, channels, sample rate, endianness, signedness)
- Provide factory methods (`Get()`) to create decoder instances for a given file
- Establish contract for file opening, decoding, rewinding, and closing operations

## External Dependencies
- `cseries.h` ΓÇö Core typedefs (`int32`, `uint8`, `float`)
- `FileHandler.h` ΓÇö `FileSpecifier` class for file references

# Source_Files/Sound/MADDecoder.cpp
## File Purpose
Implements MP3 audio decoding for the Aleph One game engine using libmad. Provides streaming decoding of MPEG audio frames into PCM samples, handling buffering, endianness conversion, and resource lifecycle.

## Core Responsibilities
- Initialize and manage libmad decoder state (stream, frame, synthesis)
- Open MP3 files and validate format (detect channels, sample rate)
- Stream decode MPEG frames into PCM samples on demand
- Buffer input data from file and manage frame boundaries
- Convert fixed-point audio samples to 16-bit integers with endianness handling
- Support stream rewinding and resource cleanup

## External Dependencies
- **libmad**: `<mad.h>` ΓÇö MPEG audio decoder library (provides Stream, Frame, Synth structs and encode/decode functions)
- **Game engine**: `FileSpecifier` (file abstraction), `StreamDecoder` (base class), `cseries.h` (common type definitions)
- **Standard library**: `<cstring>` (implicit via `memmove`); `<climits>` (implicit via `SHRT_MAX`)
- **Platform conditionals**: `ALEPHONE_LITTLE_ENDIAN` (endianness), `HAVE_MAD` (availability guard)

# Source_Files/Sound/MADDecoder.h
## File Purpose
Header for an MP3 audio decoder that wraps the libmad library. Provides a streaming decoder interface for decoding MP3 audio frames and reporting audio properties (bit depth, sample rate, stereo/mono, endianness).

## Core Responsibilities
- Open and manage MP3 file streams via `FileSpecifier`
- Decode MP3 frames into PCM audio buffers on demand
- Maintain libmad decoder state (stream, frame, synthesis buffers)
- Report audio format metadata (sample rate, channels, bit depth, endianness, bytes-per-frame)
- Support rewind and close operations for stream control
- Buffer input data with guard space for libmad's streaming requirements

## External Dependencies
- **libmad**: `mad_stream`, `mad_frame`, `mad_synth` (when `HAVE_MAD` is defined)
- **Base class**: `StreamDecoder` (Decoder.h)
- **File handling**: `FileSpecifier`, `OpenedFile` (FileHandler.h, via cseries.h)
- **Platform**: `ALEPHONE_LITTLE_ENDIAN` macro for endianness detection

# Source_Files/Sound/Mixer.cpp
## File Purpose
Implements the core audio mixing engine for the game, managing multiple simultaneous sound channels, format conversion, and output through SDL. Handles SFX, music, network audio, and resource-based sounds with real-time mixing, resampling, and volume control.

## Core Responsibilities
- Initialize/shutdown SDL audio subsystem with configurable channels and sample rate
- Maintain and update multiple playback channels (SFX, music, network, resource)
- Perform real-time audio mixing of all active channels into a single output stream
- Support various audio formats: 8/16-bit, mono/stereo, signed/unsigned, endianness-aware
- Implement pitch/sample-rate conversion during mixing via fixed-point arithmetic
- Queue and transition between consecutive sounds on channels
- Manage music playback with dynamic buffer updates
- Handle network microphone audio with volume attenuation during transmission
- Parse and play sound resources embedded in resource files
- Thread-safe audio operations via SDL_LockAudio/SDL_UnlockAudio

## External Dependencies
- **SDL audio:** `SDL_OpenAudio()`, `SDL_CloseAudio()`, `SDL_PauseAudio()`, `SDL_LockAudio()`, `SDL_UnlockAudio()`, `SDL_RWops` family, `SDL_ReadBE16()`, `SDL_ReadBE32()`, `SDL_SwapLE16()`, `SDL_SwapBE16()`, `SDL_AudioSpec`.
- **Game modules (defined elsewhere):**
  - `SoundManager::instance()->GetNetmicVolumeAdjustment()`, `IncrementChannelCallbackCount()`, `parameters.mute_while_transmitting`.
  - `Music::instance()->FillBuffer()`, `InterruptFillBuffer()` (macOS variant).
  - `SoundHeader::Load()` (from SoundManager.h).
  - Network audio: `dequeue_network_speaker_data()`, `is_sound_data_disposable()`, `release_network_speaker_buffer()` (from network_speaker_sdl.h).
  - Game state: `dynamic_world->speaking_player_index`, `local_player_index`, `game_is_networked`.
- **Includes:** `Mixer.h` (header), `interface.h` (for `strERRORS`, `badSoundChannels`).

# Source_Files/Sound/Mixer.h
## File Purpose
Audio mixing engine for Aleph One that manages multiple sound channels, combining them into a single output stream with sample rate conversion, volume control, and real-time audio callback handling. Supports sound effects, music, and network microphone audio playback.

## Core Responsibilities
- Initialize and shut down SDL audio playback with configurable sample rates and bit depths
- Buffer and queue sound data on individual channels with pitch control
- Dynamically mix multiple concurrent audio channels into mono/stereo output
- Perform linear interpolation resampling to handle pitch shifts and sample rate conversions
- Apply per-channel and master volume control with clipping protection
- Manage music, sound effects, resource audio, and network microphone channels independently
- Support audio muting during network transmission (voice activation)
- Handle sound queuing and looping transitions

## External Dependencies
- **Includes:** `SDL_endian.h` (byte swap), `cseries.h` (common types), `network_speaker_sdl.h` (network buffer descriptor), `network_audio_shared.h` (network audio constants), `map.h` (dynamic_world state), `Music.h` (Music singleton), `SoundManager.h` (SoundManager singleton)
- **External symbols:** `local_player_index`, `game_is_networked`, `dynamic_world`, `Music::instance()`, `SoundManager::instance()`, SDL audio functions (SDL_LockAudio, SDL_UnlockAudio, SDL_SwapLE16, SDL_SwapBE16)
- **Networking:** `dequeue_network_speaker_data()`, `release_network_speaker_buffer()`, `is_sound_data_disposable()` (defined elsewhere)

# Source_Files/Sound/Music.cpp
## File Purpose

Implements singleton music playback system for intro and level music. Manages audio decoding, buffering, crossfading, and playlist sequencing. Integrates with `Mixer` for final audio output and `StreamDecoder` for format handling.

## Core Responsibilities

- Singleton instance management for global music state
- Open/close/load music files via `FileSpecifier` and `StreamDecoder`
- Intro music lifecycle (setup, restart, playback)
- Level music playlist management (sequential/random playback with seeding)
- Fade-out timing and volume ramping
- Audio buffer filling from decoded streams
- Frame update (Idle) for state transitions and fade effects
- Platform-specific buffer handling (macOS interrupt-driven vs. direct update)

## External Dependencies

- **Mixer.h**: `Mixer::instance()`, `StartMusicChannel()`, `UpdateMusicChannel()`, `StopMusicChannel()`, `SetMusicChannelVolume()`, `MusicPlaying()`
- **SoundManager.h**: `SoundManager::instance()`, `IsInitialized()`, `IsActive()`, `parameters.music` (volume)
- **Decoder.h**: `StreamDecoder::Get()` factory, `Decode()`, `Rewind()`, format queries
- **FileHandler.h**: `FileSpecifier` type
- **Random.h**: `GM_Random::KISS()`, `SetTable()`
- **XML_LevelScript.h**: Included but not directly used in this file
- **SDL**: `SDL_GetTicks()`, `SDL_LockAudio()`, `SDL_UnlockAudio()`
- **cseries.h**: Type definitions (`int32`, `uint32`, `_fixed`)

# Source_Files/Sound/Music.h
## File Purpose
Defines the `Music` singleton class that manages playback of intro and level music in the Aleph One engine. Handles audio decoding, buffer management, playback control (play/pause/stop/fade), and level playlist administration with optional randomization.

## Core Responsibilities
- Music file loading and decoder initialization via `StreamDecoder`
- Playback control operations (play, pause, stop, restart, fade out)
- Audio buffer management and on-demand decoding (`FillBuffer()`)
- Intro music setup and restart
- Level music playlist creation, shuffling, and sequential/random playback
- Volume synchronization with `SoundManager`
- Platform-specific handling (macOS audio interrupts vs. SDL)

## External Dependencies
- `cseries.h`: Platform abstraction macros, SDL, fixed-point types
- `Decoder.h`: `StreamDecoder` abstract class for audio format decoders
- `FileHandler.h`: `FileSpecifier` for file paths
- `Random.h`: `GM_Random` for shuffling level playlists
- `SoundManager.h`: `SoundManager::instance()` for volume parameters
- `<vector>`: STL for `music_buffer` and `playlist`
- SDL (via cseries.h): `SDL_RWops` for file I/O

# Source_Files/Sound/ReplacementSounds.cpp
## File Purpose
Implements sound replacement management for the Aleph One engine, allowing external audio files to override in-game sounds. Manages a singleton registry of sound options indexed by sound ID and slot, with hash table acceleration.

## Core Responsibilities
- Load external audio files via decoder abstraction, extracting format metadata (bit depth, sample rate, stereo/mono, endianness)
- Maintain a singleton `SoundReplacements` registry with O(1) lookup via hash table
- Provide fallback linear search for hash misses with automatic hash table updates
- Add/update sound option entries by index and slot pair

## External Dependencies
- `Decoder.h` ΓÇô abstract decoder interface for audio decompression
- `SoundFile.h` (via include) ΓÇô `SoundHeader` base class with buffer allocation (`Load()`) and metadata fields
- `FileHandler.h` (via Decoder.h) ΓÇô `FileSpecifier` type for file I/O
- `cseries.h` (via Decoder.h) ΓÇô standard integer types and platform macros

# Source_Files/Sound/ReplacementSounds.h
## File Purpose
Manages external sound file replacements specified via MML configuration. Provides a singleton registry (`SoundReplacements`) to load and retrieve custom sound files keyed by sound index and permutation slot, with a hash table for efficient lookup.

## Core Responsibilities
- Define `ExternalSoundHeader` class for loading sound data from external files (extends `SoundHeader`)
- Store sound replacement metadata (file path, parsed header) in `SoundOptions` struct
- Implement singleton `SoundReplacements` registry with hash-based lookup by sound index and slot
- Provide hash function to avoid collisions between sounds with same index but different slots
- Support runtime addition and reset of replacement sounds

## External Dependencies
- **Includes:** `<string>`, `"SoundFile.h"` (bundled)
- **Base classes:** `SoundHeader` (from `SoundFile.h`)
- **Types used:** `std::string`, `std::vector`, `FileSpecifier` (from `FileHandler.h` via `SoundFile.h`)
- **Defined elsewhere:** `FileSpecifier`, `SoundHeader`, `uint8`, `int16`, `int32`

# Source_Files/Sound/SndfileDecoder.cpp
## File Purpose
Implements a sound file decoder using the libsndfile library. Provides functionality to open audio files, decode PCM samples into buffers, seek/rewind playback, and query audio metadata (sample rate, channels, frame count). Conditional compilation (`#ifdef HAVE_SNDFILE`) makes libsndfile support optional.

## Core Responsibilities
- Open sound files via libsndfile and initialize metadata
- Decode audio samples from file into a provided byte buffer
- Seek and rewind file playback position
- Close files and release libsndfile handles
- Expose audio properties (bit depth, stereo/mono, endianness, sample rate, frame count)
- Manage resource cleanup on destruction

## External Dependencies
- **Inherits from:** `Decoder` (defined elsewhere; abstract sound decoder interface)
- **Uses:** `FileSpecifier` class (file path abstraction, defined elsewhere)
- **Libsndfile API:** `sf_open()`, `sf_close()`, `sf_read_short()`, `sf_seek()`, `SNDFILE`, `SF_INFO`
- **Custom typedefs:** `uint8`, `int32` (assumed defined in project codebase)
- **Conditional compilation:** Entire implementation guarded by `#ifdef HAVE_SNDFILE`

# Source_Files/Sound/SndfileDecoder.h
## File Purpose
Header declaration for `SndfileDecoder`, a concrete audio decoder that wraps libsndfile. Decodes audio files (WAV, FLAC, etc.) into PCM frames for the game's sound system.

## Core Responsibilities
- Open and manage audio files via libsndfile
- Decode audio frames into a PCM buffer on demand
- Query audio format metadata (channels, bit depth, sample rate, endianness)
- Manage file lifecycle (open, rewind, close)
- Implement the `Decoder` abstract interface for polymorphic audio handling

## External Dependencies
- `"Decoder.h"` ΓÇö base class `Decoder` and type `FileSpecifier`
- `"sndfile.h"` ΓÇö libsndfile C library (conditional on `HAVE_SNDFILE` macro)
- Indirectly from `Decoder.h`: `cseries.h` (defines `uint8`, `int32`), `FileHandler.h` (file abstraction)

# Source_Files/Sound/song_definitions.h
## File Purpose
Header file defining the data structures and constants for song/music definitions in the game engine. Specifies how songs are structured with introduction, chorus, and trailer segments. Provides a table of song metadata for the game's audio system.

## Core Responsibilities
- Define the `sound_snippet` structure (offset-based audio segment representation)
- Define the `song_definition` structure (song metadata with playback control)
- Define song behavior flags (`_song_automatically_loops`)
- Provide macro for encoding random chorus counts (`RANDOM_COUNT`)
- Declare a global array of song definitions for engine initialization

## External Dependencies
- `MACHINE_TICKS_PER_SECOND`: macro constant (defined elsewhere, platform-specific timing)
- Standard C integer types: `int16`, `int32` (from platform headers)


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

# Source_Files/Sound/SoundFile.cpp
## File Purpose
Implements sound file parsing, loading, and management for the Aleph One game engine. Handles System 7 sound headers, sound definitions with multiple permutations, and integration with external audio decoders for custom sounds.

## Core Responsibilities
- Parse and validate System 7 sound header formats (standard and extended variants)
- Load audio data from in-memory buffers or file streams
- Manage hierarchical sound organization (sources ΓåÆ definitions ΓåÆ permutations)
- Support lazy loading of sound permutations
- Integrate external audio files via decoder abstraction
- Provide custom sound slot allocation and management
- Track and report audio properties (channels, bit depth, sample rate, loop points)

## External Dependencies

- **Logging.h:** `logWarning3()` for diagnostic messages
- **csmisc.h:** `machine_tick_count()` for timestamp
- **Decoder.h:** `Decoder::Get()`, `StreamDecoder` interface for external audio decoding
- **FileHandler.h:** `FileSpecifier`, `OpenedFile` for file I/O
- **AStream.h:** `AIStreamBE` (big-endian binary stream reader, "defined elsewhere")
- **SoundFile.h:** Type definitions and public API
- **Standard Library:** `<vector>`, `<memory>` (auto_ptr), `<boost/shared_array.hpp>`
- **Macros:** `FOUR_CHARS_TO_INT()`, `FIXED_ONE` (defined elsewhere)

# Source_Files/Sound/SoundFile.h
## File Purpose
Defines core sound file management classes for the Aleph One game engine. Provides abstractions for loading, parsing, and accessing sound resources from System 7 sound format files, including support for sound permutations, pitch variation, and runtime custom sound loading.

## Core Responsibilities
- Parse and deserialize System 7 sound format headers (standard and extended variants)
- Manage individual sound sample data with metadata (bit depth, stereo, endianness, loop points, playback rate)
- Organize sounds into definitions with multiple permutations and probabilistic playback selection
- Maintain a hierarchical sound file structure indexed by source and sound index
- Support runtime custom sound addition without modifying the base sound file

## External Dependencies
- `AStream.h`: Endian-aware deserialization (uses `AIStreamBE` for big-endian System 7 parsing)
- `FileHandler.h`: Cross-platform file I/O (`OpenedFile`, `FileSpecifier`)
- `<memory>`, `<vector>`: C++ standard library
- `<boost/shared_array.hpp>`: Boost smart array container for sample data
- `cstypes.h`: Custom integer types (`uint8`, `int16`, `int32`, `uint16`, `uint32`, `_fixed`) ΓÇö defined elsewhere

# Source_Files/Sound/SoundManager.cpp
## File Purpose
Core implementation of the SoundManager singleton that controls all audio in the engine. Handles sound playback, 3D positioning, volume control, memory management, ambient sounds, and integration with the low-level Mixer for SDL audio output.

## Core Responsibilities
- **Initialization & lifecycle** ΓÇö Initialize sound system with parameters, open/close sound files, shutdown on exit via atexit handler
- **Sound playback control** ΓÇö Play, stop, and manage individual sounds with optional 3D positioning and pitch modulation
- **Volume & parameter management** ΓÇö Adjust master volume, test volume levels, configure stereo/dynamic-tracking/ambient flags
- **Channel allocation** ΓÇö Select best available channel for sounds based on priority and volume, free channels when unused
- **Memory management** ΓÇö Load/unload sound data into fixed-size buffers, evict least-recently-used sounds on overflow
- **3D audio** ΓÇö Calculate stereo volumes based on listener/source positions, handle distance-based attenuation and obstruction
- **Ambient sounds** ΓÇö Update looping environmental sounds (water, machinery, wind), manage up to 4 concurrent ambient channels
- **Custom sounds** ΓÇö Support runtime-defined sound slots and MML-based external sound replacements

## External Dependencies
- **Includes/Imports:**
  - `SoundManager.h` ΓÇö class definition and Parameters/Channel structs
  - `ReplacementSounds.h` ΓÇö `SoundReplacements::instance()` for external sound replacement lookup
  - `sound_definitions.h` ΓÇö `sound_behavior_definition`, `ambient_sound_definition`, `random_sound_definition`, global sound behavior curves
  - `Mixer.h` ΓÇö `Mixer::instance()` for low-level audio output
  - `world.h` ΓÇö `world_location3d`, `world_distance` types for 3D positioning
  - `XML_ElementParser.h` ΓÇö base class for XML configuration parsers
  - `FileHandler.h` ΓÇö `FileSpecifier` for file I/O
  - `SoundFile.h` ΓÇö `SoundFile` class for loading sound resources
- **External symbols/callbacks (defined elsewhere):**
  - `_sound_listener_proc()` ΓÇö retrieves listener location and facing each frame
  - `_sound_obstructed_proc(source)` ΓÇö queries obstruction flags (obstructed, media-obstructed, media-muffled)
  - `_sound_add_ambient_sources_proc()` ΓÇö callback to accumulate ambient sound sources
  - Game-engine types: `angle`, `_fixed`, `machine_tick_count()`, `local_random()`, `distance3d()`

# Source_Files/Sound/SoundManager.h
## File Purpose
Singleton manager class for all sound playback in the Aleph One game engine. Handles sound file loading, real-time audio channel allocation, spatial audio (3D positioning, attenuation, panning), ambient sound processing, and volume control. Acts as the central audio subsystem entry point.

## Core Responsibilities
- **Singleton lifecycle**: Initialize, configure parameters, and shutdown the audio system
- **Sound file I/O**: Open/close external sound definition files and manage custom sound definitions
- **Audio playback**: Play sounds with 3D world positioning, stop playing sounds, query playback status
- **Channel management**: Allocate and free audio channels (~32 simultaneous), track active sounds per channel
- **Spatial audio**: Calculate volume, pitch shift, and stereo panning based on sound source position and listener location
- **Ambient sounds**: Track and update looping ambient sound sources in the game world
- **Volume control**: Adjust master volume, music volume, voice ducking during mic transmission
- **Sound resource management**: Load/unload sound definitions, orphan/release cached sounds, manage sound permutations

## External Dependencies

- **cseries.h**: Common utilities, fixed-point math (`_fixed`), standard defines
- **FileHandler.h**: `FileSpecifier`, `OpenedFile` (file I/O abstraction)
- **SoundFile.h**: `SoundDefinition`, `SoundFile` (sound format parsing and caching)
- **world.h**: `world_location3d`, `angle`, `world_distance` (3D positioning, angles), trig tables
- **XML_ElementParser.h**: `XML_ElementParser` (parser for sound definitions from XML)
- **SoundManagerEnums.h**: Sound codes (`_snd_*`), ambient sound codes, initialization flags, volume enums

**External Functions** (defined elsewhere):
- `_sound_listener_proc()` ΓåÆ returns current listener position and facing
- `_sound_obstructed_proc()` ΓåÆ queries occlusion/media state between source and listener
- `_sound_add_ambient_sources_proc()` ΓåÆ callback to register ambient sound sources
- `Sounds_GetParser()` ΓåÆ returns XML parser for \<sounds\> elements
- Sound accessor functions: `Sound_TerminalLogon()`, `Sound_GotPowerup()`, etc. ΓåÆ hardcoded sound IDs (legacy)

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

## External Dependencies
- **FIXED_ONE** macro (defined elsewhere) ΓÇö used in frequency constants for fixed-point arithmetic (`_lower_frequency`, `_normal_frequency`, `_higher_frequency`)
- Consumed by: SoundManager.h/SoundManager.cpp and audio subsystem callers


# Source_Files/Sound/VorbisDecoder.cpp
## File Purpose
Implements the `VorbisDecoder` class, which decodes Ogg Vorbis audio files using libvorbisfile. Bridges SDL file I/O (`SDL_RWops`) with the Vorbis decoding API to enable streaming audio playback in the Aleph One game engine.

## Core Responsibilities
- Wrap SDL file operations (read, seek, close, tell) as callbacks for libvorbisfile
- Open and validate Ogg Vorbis files, extracting audio format metadata (channels, sample rate)
- Decode compressed Vorbis data into PCM audio buffers on demand
- Manage file position (rewind to start)
- Resource cleanup on decoder destruction

## External Dependencies
- **Vorbis/libvorbisfile:** `ov_test_callbacks`, `ov_test_open`, `ov_info`, `ov_read`, `ov_clear`, `ov_raw_seek`, `OggVorbis_File`, `ov_callbacks`, `vorbis_info`
- **SDL:** `SDL_RWops`, `SDL_RWread`, `SDL_RWseek`, `SDL_RWclose`, `SDL_RWtell`
- **Base class:** `StreamDecoder` (defined elsewhere in header)
- **Utilities:** `FileSpecifier`, `IsLittleEndian()`, type aliases `int32`, `uint8` (from `cseries.h`)

# Source_Files/Sound/VorbisDecoder.h
## File Purpose
Defines the `VorbisDecoder` class that handles decoding and streaming of OGG Vorbis audio files. It inherits from `StreamDecoder` and wraps the libvorbis library to provide a unified interface for audio decoding within the game engine's sound system.

## Core Responsibilities
- Open and close OGG Vorbis audio files via `FileSpecifier`
- Decode Vorbis-compressed audio data into raw PCM samples
- Rewind playback position to the beginning of a file
- Query audio format metadata (bit depth, channels, sample rate, endianness, frame size)
- Manage underlying `OggVorbis_File` handle and vorbis callbacks

## External Dependencies
- `cseries.h` ΓÇö provides common type definitions (`uint8`, `int32`, `float`) and utilities
- `Decoder.h` ΓÇö defines parent class `StreamDecoder` and `FileSpecifier` (indirectly)
- `<vorbis/vorbisfile.h>` ΓÇö libvorbis public API (conditional)
- `OggVorbis_File`, `ov_callbacks` ΓÇö types from libvorbis (external symbols)


