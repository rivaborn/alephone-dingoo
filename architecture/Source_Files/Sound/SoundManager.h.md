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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Parameters | struct | Audio configuration: sample rate, buffer size, channel count, volume levels, flags (stereo, dynamic tracking, Doppler shift, 16-bit audio) |
| Channel | struct | Single audio channel state: flags, sound_index, identifier, volume/pitch variables, dynamic source location, mixer channel reference, callback count |
| Pause | class | RAII wrapper to temporarily pause/unpause audio (constructor pauses, destructor resumes) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| m_instance | SoundManager* | static (singleton) | Singleton instance pointer |
| parameters | Parameters | member | Current audio configuration |
| channels | std::vector\<Channel\> | member | Pool of audio channels |
| sound_file | SoundFile | member | Loaded sound definition file |
| initialized | bool | member | Has Initialize() completed? |
| active | bool | member | Is audio engine currently active? |
| MAXIMUM_SOUND_CHANNELS | int | static const | Max simultaneous sounds (32) |
| MAXIMUM_AMBIENT_SOUND_CHANNELS | int | static const | Max ambient sources (4) |
| DEFAULT_SOUND_LEVEL, DEFAULT_MUSIC_LEVEL | int | static const | Init volume levels (platform-specific) |

## Key Functions / Methods

### instance()
- **Signature**: `static inline SoundManager* instance()`
- **Purpose**: Lazy-initialize and return singleton instance
- **Inputs**: None
- **Outputs/Return**: Pointer to SoundManager singleton
- **Side effects**: Allocates new SoundManager if not yet created
- **Notes**: Not thread-safe; assumes single-threaded access

### Initialize(const Parameters&)
- **Signature**: `void Initialize(const Parameters&)`
- **Purpose**: Set up audio subsystem with given configuration
- **Inputs**: Parameters struct (sample rate, channels, flags, volumes)
- **Outputs/Return**: None
- **Side effects**: Sets `initialized=true`, configures audio backend, creates channel pool
- **Calls**: (implementation elsewhere)
- **Notes**: Must be called before playback; expected during engine startup

### PlaySound(short, world_location3d*, short, _fixed)
- **Signature**: `void PlaySound(short sound_index, world_location3d *source, short identifier, _fixed pitch)`
- **Purpose**: Play a sound at a 3D world location with optional pitch modification
- **Inputs**: `sound_index` (sound ID), `source` (world position; NULL for immobile), `identifier` (owner/grouping ID), `pitch` (playback speed multiplier)
- **Outputs/Return**: None
- **Side effects**: Allocates a Channel, loads sound, calculates spatial audio variables (volume, pan, pitch shift), triggers mixer
- **Calls**: `BestChannel()`, `CalculateSoundVariables()`, `BufferSound()`, `GetSoundDefinition()`
- **Notes**: 3D sounds use source location and listener location (from `_sound_listener_proc()`) to apply attenuation and panning; immobile sounds (source==NULL) skips spatial calculations

### PlayLocalSound(short, _fixed)
- **Signature**: `void PlayLocalSound(short sound_index, _fixed pitch)`
- **Purpose**: Play a sound without 3D positioning (listener-relative)
- **Inputs**: `sound_index`, `pitch`
- **Outputs/Return**: None
- **Side effects**: Calls `PlaySound(sound_index, NULL, NONE, pitch)` internally
- **Notes**: Convenience wrapper; equivalent to playing at listener position with no attenuation

### StopSound(short, short)
- **Signature**: `void StopSound(short identifier, short sound_index)`
- **Purpose**: Stop playback of a specific sound or all sounds matching criteria
- **Inputs**: `identifier` (NONE=any), `sound_index` (NONE=any)
- **Outputs/Return**: None
- **Side effects**: Frees affected channels, marks sounds as no-longer-playing
- **Calls**: `FreeChannel()`
- **Notes**: `StopAllSounds()` passes NONE/NONE to stop everything

### LoadSound(short)
- **Signature**: `bool LoadSound(short sound)`
- **Purpose**: Load sound data into memory from the open sound file
- **Inputs**: `sound` (sound definition index)
- **Outputs/Return**: Success
- **Side effects**: Allocates memory, may evict least-useful sounds if buffer is full
- **Calls**: `GetSoundDefinition()`, `ReleaseLeastUsefulSound()`
- **Notes**: Lazy-load model; actual playback may trigger additional loading

### Idle()
- **Signature**: `void Idle()`
- **Purpose**: Per-frame update for dynamic audio processing
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Updates listener-source distances, recalculates spatial audio for active channels, updates ambient sound positions, applies Doppler shift if enabled
- **Calls**: `UpdateAmbientSoundSources()`, `TrackStereoSounds()`, `_sound_listener_proc()`
- **Notes**: Must be called regularly (ideally each game frame) to maintain active spatial audio; respects `_dynamic_tracking_flag` in parameters

### GetNetmicVolumeAdjustment()
- **Signature**: `inline int16 GetNetmicVolumeAdjustment()`
- **Purpose**: Return volume reduction applied during network voice transmission
- **Inputs**: None
- **Outputs/Return**: Volume delta (typically MAXIMUM_SOUND_VOLUME / 8)
- **Side effects**: None
- **Notes**: Query-only; used by voice/netmic systems to duck game audio

### BestChannel(short, Channel::Variables&) [private]
- **Signature**: `Channel* BestChannel(short sound_index, Channel::Variables& variables)`
- **Purpose**: Find an available or low-priority channel to play a new sound
- **Inputs**: `sound_index` (to check priority), `variables` (volume/pitch to compare)
- **Outputs/Return**: Pointer to Channel to use (may evict lower-priority sound)
- **Side effects**: Frees a lower-priority channel if all are in use; increments callback count
- **Notes**: Implements priority/amplitude thresholding; compares against `ABORT_AMPLITUDE_THRESHHOLD`

### CalculateSoundVariables(short, world_location3d*, Channel::Variables&) [private]
- **Signature**: `void CalculateSoundVariables(short sound_index, world_location3d *source, Channel::Variables& variables)`
- **Purpose**: Compute volume and stereo panning based on 3D source position and listener location
- **Inputs**: `sound_index`, 3D source position (or NULL), output structure
- **Outputs/Return**: None (fills `variables.left_volume`, `variables.right_volume`, `variables.volume`)
- **Side effects**: Queries `_sound_listener_proc()`, `_sound_obstructed_proc()` for listener state and occlusion
- **Calls**: `AngleAndVolumeToStereoVolume()`
- **Notes**: Respects sound obstructing logic (walls, different media); applies volume ducking if in different media from listener

## Control Flow Notes

**Initialization & Shutdown (one-time)**:
- `Initialize()` ΓåÆ audio backend setup, channel pool creation
- `Shutdown()` ΓåÆ cleanup and deallocation

**Per-Frame Update Loop** (game tick):
- Game calls `Idle()` on SoundManager
- `Idle()` recalculates spatial audio for all active channels via `UpdateAmbientSoundSources()`, `TrackStereoSounds()`
- Channels update their output to audio backend

**Sound Playback** (on-demand):
- Game calls `PlaySound()` with 3D location
- `PlaySound()` ΓåÆ `BestChannel()` (find slot) ΓåÆ `CalculateSoundVariables()` (compute audio) ΓåÆ `BufferSound()` (upload to channel) ΓåÆ audio backend plays
- Can be interrupted by `StopSound()`

**Resource Management**:
- `LoadSound()` loads data; `UnloadAllSounds()` flushes
- `OrphanSound()` marks as releasable; `ReleaseLeastUsefulSound()` evicts oldest/quietest

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
