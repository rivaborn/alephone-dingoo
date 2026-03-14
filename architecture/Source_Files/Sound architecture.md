# Subsystem Overview

## Purpose

The Sound subsystem manages all audio playback in the Aleph One engine, including music, sound effects, 3D spatial audio positioning, ambient environmental sounds, and network microphone data. It provides a unified interface through the `SoundManager` singleton while delegating low-level audio output to SDL and supporting multiple compressed audio formats via pluggable decoders.

## Key Files

| File | Role |
|------|------|
| SoundManager.cpp / SoundManager.h | Core singleton managing all sound playback, channel allocation, 3D audio positioning, ambient sounds, and memory management |
| Mixer.cpp / Mixer.h | Real-time audio mixing engine that combines multiple simultaneous channels into SDL output with resampling and volume control |
| Music.cpp / Music.h | Singleton music playback system managing intro and level music with decoder integration, crossfading, and playlist sequencing |
| SoundFile.cpp / SoundFile.h | Parses and manages System 7 sound format files with support for permutations, custom sounds, and lazy loading |
| Decoder.cpp / Decoder.h | Factory pattern for audio decoders; instantiates appropriate decoder based on file type |
| BasicIFFDecoder.cpp / BasicIFFDecoder.h | Decodes uncompressed AIFF and WAV audio files with metadata extraction and buffered frame access |
| MADDecoder.cpp / MADDecoder.h | MP3 decoder wrapping libmad library with MPEG frame streaming and sample conversion |
| VorbisDecoder.cpp / VorbisDecoder.h | OGG Vorbis decoder wrapping libvorbisfile with SDL file I/O integration |
| SndfileDecoder.cpp / SndfileDecoder.h | Optional libsndfile decoder supporting WAV, FLAC, and other formats (conditional `HAVE_SNDFILE`) |
| ReplacementSounds.cpp / ReplacementSounds.h | Singleton registry for external sound file replacements with hash-table acceleration for O(1) lookup |
| sound_definitions.h | Core enumerations and structures defining sound behavior, volume curves, ambient/random sound indices |
| song_definitions.h | Data structures for song metadata with intro/chorus/trailer segments and playback control flags |
| SoundManagerEnums.h | Enumeration constants for sound effect codes, ambient sound codes, and audio configuration flags |

## Core Responsibilities

- Initialize and shut down SDL audio subsystem with configurable sample rates, bit depths, and channel counts
- Load and decode audio files in multiple formats (WAV, AIFF, MP3 via libmad, OGG Vorbis, libsndfile) via pluggable decoder abstraction
- Mix multiple concurrent audio channels in real-time using fixed-point arithmetic for pitch and sample-rate conversion
- Calculate 3D spatial audio with distance-based attenuation, stereo panning, and obstruction-aware volume falloff based on listener and source positions
- Manage sound definitions with permutations, probabilistic selection, and depth-based volume curves
- Handle music playback with audio decoding, buffer management, crossfading, and level-based playlist sequencing
- Support external sound replacement via MML configuration with hash-accelerated registry lookup
- Update and manage up to 4 concurrent ambient environmental sounds (water, machinery, wind) with looping
- Allocate and prioritize audio channels for competing sounds using LRU cache and priority-based eviction
- Apply master volume, per-channel volume, and voice ducking (attenuation during network transmission)

## Key Interfaces & Data Flow

**Exposes to callers:**
- `SoundManager::instance()` singleton with public methods: `Initialize()`, `Shutdown()`, `PlaySound()`, `StopSound()`, `SetVolume()`, `IsInitialized()`, `IsActive()`
- `Music::instance()` for music playback: `Play()`, `Stop()`, `Pause()`, `FillBuffer()`, `MusicPlaying()`
- `Mixer::instance()` for low-level channel control (internal use)
- Sound file loading and resource management via `FileSpecifier` abstraction

**Consumes from other subsystems:**
- **FileHandler:** `FileSpecifier`, `OpenedFile` for audio file I/O
- **Game state:** `local_player_index`, `dynamic_world` (current world state), `game_is_networked` for context-aware behavior
- **Network audio:** `dequeue_network_speaker_data()`, `release_network_speaker_buffer()`, `is_sound_data_disposable()` for microphone voice playback
- **Random number generation:** `GM_Random::KISS()` for playlist shuffling and probabilistic sound selection
- **Callbacks (defined by game engine):** `_sound_listener_proc()`, `_sound_obstructed_proc()`, `_sound_add_ambient_sources_proc()` for 3D positioning and environment queries
- **Logging:** `logWarning3()` for diagnostic output
- **Fixed-point math:** `_fixed` type and `FIXED_ONE` constant

## Runtime Role

**Initialization phase:** `SoundManager::Initialize()` opens sound resource files via `FileSpecifier`, initializes the `Mixer` which calls `SDL_OpenAudio()` with configured `SDL_AudioSpec`, and sets up the audio callback handler that invokes `Mixer::FillBuffer()`.

**Per-frame updates:** `Music::Idle()` updates music state (fade transitions, buffer refills, playlist advancement); `SoundManager` maintains active channels and updates ambient sound sources via the registered callback; audio mixing occurs asynchronously in the SDL audio callback thread.

**Shutdown phase:** `SoundManager::Shutdown()` stops all active sounds, closes music decoders, and calls `SDL_CloseAudio()` to clean up SDL audio resources; registered `atexit()` handler ensures cleanup on engine exit.

## Notable Implementation Details

- **Multi-decoder factory pattern:** `Decoder::Get()` tries decoders in priority order (SndfileDecoder ΓåÆ BasicIFFDecoder ΓåÆ VorbisDecoder ΓåÆ MADDecoder), returning the first successful match to support conditional compilation of expensive codecs (libmad, libvorbis, libsndfile may not be available on constrained platforms).
- **Format agnostic mixing:** Mixer handles 8/16-bit, mono/stereo, signed/unsigned, and endian-aware samples by normalizing to a working format (16-bit signed) and applying SDL byte-swap macros (`SDL_SwapLE16()`, `SDL_SwapBE16()`) during conversion.
- **Fixed-point resampling:** Pitch shifts and sample-rate conversions during mixing use fixed-point arithmetic (`_fixed` type) with linear interpolation to avoid floating-point overhead on constrained MIPS hardware.
- **Thread-safe audio updates:** Critical sections use `SDL_LockAudio()` / `SDL_UnlockAudio()` to serialize access to channel state during real-time audio callback execution.
- **Sound cache with LRU eviction:** SoundManager maintains a bounded buffer pool for loaded sound data; when capacity is exceeded, least-recently-used sounds are unloaded to make room for new ones.
- **Lazy permutation loading:** Sound definitions can have multiple permutations; permutation data is loaded on-demand rather than all at initialization, reducing memory footprint.
- **Ambient sound callback integration:** Game engine registers `_sound_add_ambient_sources_proc()` to dynamically inject ambient sound sources into the update loop each frame, decoupling audio from world state queries.
- **Network voice ducking:** When `game_is_networked` and local player is transmitting, `parameters.mute_while_transmitting` reduces non-voice audio volume to avoid mic feedback.
- **3D audio obstruction modeling:** Sound volume and attenuation are computed using callbacks that query obstruction flags (obstructed, media-obstructed, media-muffled) between listener and source, allowing environmental effects.
