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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SoundManager::Parameters | struct | Global sound settings: channel count, volume, sample rate, flags (stereo, ambient, 16-bit) |
| SoundManager::Channel | struct | Active sound instance: playback state, position, pitch, volume, source location |
| SoundManager::Channel::Variables | nested struct | Per-channel audio variables: priority, volume, left/right stereo volumes, pitch |
| ambient_sound_data | struct (local) | Temporary ambient sound slot: flags, sound index, computed variables, channel pointer |
| SoundDefinition | (external) | Sound file metadata: behavior, pitch range, permutations, loaded data pointer |
| ambient_sound_definition | struct | Minimal mapping: sound_index only; parsed from XML |
| random_sound_definition | struct | Minimal mapping: sound_index only; parsed from XML |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| real_number_of_sound_definitions | int16 | static | Counts original sound definitions before custom sounds added |
| m_instance | SoundManager* | static member | Singleton instance pointer |
| _Sound_TerminalLogon, _Sound_TerminalLogoff, ... | short | static | Hardcoded sound indices that can be overridden via XML |
| original_ambient_sound_definitions | ambient_sound_definition* | static | Backup of original ambient definitions for XML reset |
| original_random_sound_definitions | random_sound_definition* | static | Backup of original random definitions for XML reset |
| AmbientSoundAssignParser, RandomSoundAssignParser, ... | XML_*Parser | static | XML configuration parsers instantiated at module level |

## Key Functions / Methods

### Initialize
- **Signature:** `void Initialize(const Parameters& new_parameters)`
- **Purpose:** Set up sound system, open default sound file, configure parameters, register shutdown handler
- **Inputs:** Parameters struct (channels, volume, flags, rate, samples, music level)
- **Outputs/Return:** None
- **Side effects:** Opens sound file, calls `atexit(Shutdown)`, transitions `initialized` and `active` flags
- **Calls:** `OpenSoundFile`, `SetParameters`, `SetStatus`
- **Notes:** Only executes if file opens successfully; uses singleton pattern

### SetParameters
- **Signature:** `void SetParameters(const Parameters& parameters)`
- **Purpose:** Update runtime audio configuration (unload/reload with new settings)
- **Inputs:** New Parameters struct
- **Outputs/Return:** None
- **Side effects:** Toggles sound system off/on, unloads all sounds, updates `parameters` and calls `Verify()`
- **Calls:** `SetStatus`, `UnloadAllSounds`
- **Notes:** Preserves active state if volume > 0

### PlaySound
- **Signature:** `void PlaySound(short sound_index, world_location3d *source, short identifier, _fixed pitch)`
- **Purpose:** Play a sound at a 3D location with optional pitch modifier
- **Inputs:** sound index, source location (NULL for local), identifier for orphaning, pitch multiplier
- **Outputs/Return:** None
- **Side effects:** Loads sound into memory, selects/configures a channel, marks channel as used, starts playback via Mixer
- **Calls:** `CalculateInitialSoundVariables`, `LoadSound`, `BestChannel`, `InstantiateSoundVariables`, `BufferSound`
- **Notes:** Returns silently if inactive, no channels, bad index, or zero volume; pitch modifier is combined with sound definition's intrinsic pitch range

### DirectPlaySound
- **Signature:** `void DirectPlaySound(short sound_index, angle direction, short volume, _fixed pitch)`
- **Purpose:** Play sound with direct angle/volume, bypassing 3D calculation
- **Inputs:** sound index, direction angle, explicit volume, pitch
- **Outputs/Return:** None
- **Side effects:** Loads sound, selects channel, applies stereo panning based on direction
- **Calls:** `LoadSound`, `BestChannel`, `AngleAndVolumeToStereoVolume`, `InstantiateSoundVariables`, `BufferSound`
- **Notes:** Treats sound as local if direction is NONE or listener unavailable

### LoadSound
- **Signature:** `bool LoadSound(short sound_index)`
- **Purpose:** Ensure sound data is resident in memory, loading from file or using external replacements
- **Inputs:** sound index
- **Outputs/Return:** true if successfully loaded and has data, false otherwise
- **Side effects:** Calls `definition->Load()`, updates `loaded_sounds_size`, evicts least-used sounds if buffer full
- **Calls:** `GetSoundDefinition`, `SoundReplacements::GetSoundOptions`, `definition->Load()`, `ReleaseLeastUsefulSound`
- **Notes:** Handles ambient sound flag filtering; respects external sound replacements; manages LRU buffer

### Idle
- **Signature:** `void Idle()`
- **Purpose:** Per-frame maintenance: update 3D sound tracking, manage ambient sound sources
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Unlocks finished sounds, updates stereo pans for moving sources, triggers ambient sound updates
- **Calls:** `UnlockLockedSounds`, `TrackStereoSounds`, `CauseAmbientSoundSourceUpdate`
- **Notes:** Called once per frame; no-op if inactive

### BestChannel
- **Signature:** `Channel *BestChannel(short sound_index, Channel::Variables &variables)`
- **Purpose:** Select or reuse a channel for a new sound
- **Inputs:** sound index to play, pre-calculated Variables (volume, priority)
- **Outputs/Return:** Pointer to Channel struct, or NULL if no channel available
- **Side effects:** May call `FreeChannel()` to evict lower-priority/lower-volume sound
- **Calls:** `GetSoundDefinition`, `FreeChannel`, `Mixer::ChannelBusy`
- **Notes:** Respects sound's chance (probability) flag; prioritizes reusing channels playing same sound; uses `ABORT_AMPLITUDE_THRESHHOLD` to decide eviction

### CalculateSoundVariables
- **Signature:** `void CalculateSoundVariables(short sound_index, world_location3d *source, Channel::Variables& variables)`
- **Purpose:** Compute 3D audio parameters (volume, stereo panning) from listener/source geometry
- **Inputs:** sound index, source location, reference to Variables struct
- **Outputs/Return:** None; fills Variables with computed priority, volume, left_volume, right_volume
- **Side effects:** Calls listener callback, distance/angle calculations
- **Calls:** `GetSoundDefinition`, `_sound_listener_proc`, `distance3d`, `distance_to_volume`, `AngleAndVolumeToStereoVolume`, `_sound_obstructed_proc`
- **Notes:** Uses long-distance-friendly int32 math for delta calculations; if no source, sets max volumes

### UpdateAmbientSoundSources
- **Signature:** `void UpdateAmbientSoundSources()`
- **Purpose:** Manage looping ambient sounds: accumulate active sources, select top-priority, allocate ambient channels
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies ambient_sounds array and channel states; calls Mixer to buffer new ambient sounds
- **Calls:** `_sound_add_ambient_sources_proc`, `AddOneAmbientSoundSource`, `FreeChannel`, `LoadSound`, `BufferSound`, `InstantiateSoundVariables`
- **Notes:** Maintains at most `MAXIMUM_AMBIENT_SOUND_CHANNELS` (4) concurrent ambient sounds; culls lowest-priority when overflow

### AngleAndVolumeToStereoVolume
- **Signature:** `void AngleAndVolumeToStereoVolume(angle delta, short volume, short *right_volume, short *left_volume)`
- **Purpose:** Convert angle-relative-to-listener into stereo left/right volumes
- **Inputs:** delta angle, mono volume
- **Outputs/Return:** left_volume, right_volume pointers filled with panned values
- **Side effects:** None
- **Calls:** None
- **Notes:** If mono flag set, left == right == volume; uses 4-quadrant panning with fractional interpolation

### StopSound
- **Signature:** `void StopSound(short identifier, short sound_index)`
- **Purpose:** Stop playing sounds by identifier, sound index, or both (NONE wildcards)
- **Inputs:** identifier (object ID or NONE for any), sound_index (or NONE for any)
- **Outputs/Return:** None
- **Side effects:** Marks matching channels as free, optionally freezes ambient channels
- **Calls:** `FreeChannel`
- **Notes:** If both identifier and sound_index are NONE, stops all; special handling for ambient flag

---

## Control Flow Notes
- **Init/Shutdown:** `Initialize()` ΓåÆ `OpenSoundFile()` ΓåÆ `SetStatus(true)` ΓåÆ registers `atexit(Shutdown)`; `Shutdown()` stops mixer and closes file
- **Frame update:** Each frame, caller invokes `Idle()`, which calls `TrackStereoSounds()` to update 3D pans and `UpdateAmbientSoundSources()` to manage ambient channels
- **Sound playback:** `PlaySound()` / `DirectPlaySound()` ΓåÆ `LoadSound()` ΓåÆ `BestChannel()` ΓåÆ `BufferSound()` ΓåÆ `Mixer::BufferSound()`; Mixer callback handles actual mixing
- **Memory pressure:** When `loaded_sounds_size > total_buffer_size`, `ReleaseLeastUsefulSound()` evicts least-recently-played sound

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
