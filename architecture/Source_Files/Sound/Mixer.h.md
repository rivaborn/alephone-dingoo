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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Header | struct | Audio metadata: format flags (8/16-bit, mono/stereo/signed), data pointers, loop boundaries, sample rate as fixed-point |
| Channel | struct | Per-channel playback state: format info, data pointers, position counter, volumes (left/right), next queued header |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| m_instance | static Mixer* | static | Singleton pattern for global mixer access |
| local_player_index | short | extern | Current player index; used to mute audio while transmitting |
| game_is_networked | bool | extern | Flag to enable network audio muting logic |

## Key Functions / Methods

### Start
- Signature: `void Start(uint16 rate, bool sixteen_bit, bool stereo, int num_channels, int volume, uint16 samples)`
- Purpose: Initialize SDL audio subsystem and allocate channels
- Inputs: Sample rate (Hz), bit depth/format flags, channel count, master volume, buffer size
- Outputs/Return: None
- Side effects: Allocates `channels` vector, opens SDL audio device
- Calls: SDL_OpenAudio (indirect via desired/obtained specs)
- Notes: Called at engine startup; configures the audio hardware parameters

### BufferSound
- Signature: `void BufferSound(int channel, const Header& header, _fixed pitch)`
- Purpose: Queue a sound on a specific channel with playback pitch
- Inputs: Channel index, sound metadata (Header), pitch as fixed-point (0x10000 = normal)
- Outputs/Return: None
- Side effects: Activates channel, resets playback position counter
- Calls: `Channel::LoadSoundHeader`
- Notes: Immediately starts playback if channel was idle; can queue next sound while one plays

### SetChannelVolumes
- Signature: `void SetChannelVolumes(int channel, int16 left, int16 right)`
- Purpose: Set stereo volume levels for a specific channel
- Inputs: Channel index, left/right volume (0x100 = nominal)
- Outputs/Return: None (inline setter)
- Side effects: Updates channel volume state

### Mix (template)
- Signature: `template <class T, bool stereo, bool is_signed> inline void Mix(T *p, int len)`
- Purpose: Core real-time audio mixing callback; generates one audio frame from all active channels
- Inputs: Output buffer pointer (typed as 8-bit or 16-bit), frame count
- Outputs/Return: Writes to `p` buffer directly
- Side effects: Advances playback pointers, handles loop transitions, dequeues network audio buffers, clipping/normalization
- Calls: `SoundManager::instance()->GetNetmicVolumeAdjustment()`, `Music::instance()->FillBuffer()` / `InterruptFillBuffer()`, `dequeue_network_speaker_data()`, `is_sound_data_disposable()`
- Notes: 
  - Per-sample processing: reads current + next frame from each active channel, interpolates based on fractional counter, mixes to int32, applies volumes, clips to int16, converts to output format
  - Handles endianness for 16-bit samples (SDL_SwapLE16/BE16)
  - Network audio channels reduce all other channels' volume (netmic ducking)
  - Mutes audio during network transmission if `mute_while_transmitting` flag set
  - Automatically transitions to loop or dequeues next queued header when sound finishes
  - Stateless per-frame; relies on Channel state for persistence

### MusicPlaying / StopMusicChannel
- Signature: `bool MusicPlaying()` / `void StopMusicChannel()`
- Purpose: Query and control music channel playback
- Side effects: StopMusicChannel uses SDL_LockAudio for thread safety

### EnsureNetworkAudioPlaying / StopNetworkAudio
- Signature: `void EnsureNetworkAudioPlaying()` / `void StopNetworkAudio()`
- Purpose: Manage network microphone audio queuing and lifecycle
- Notes: Dequeues buffers from shared queue; buffers marked disposable are freed after playback

## Control Flow Notes
**Audio pipeline:** SDL calls `MixerCallback` (static) ΓåÆ `Callback` (instance method) ΓåÆ `Mix<>` template with appropriate type parameters based on audio format.  
**Per-frame processing:** Mix iterates all channels, reads resampled samples, accumulates into master left/right, applies master volume + muting logic, outputs clipped/formatted samples.  
**Sound lifecycle:** BufferSound ΓåÆ active playback ΓåÆ length decrements ΓåÆ loop check (restart data pointer) or next header check (dequeue) ΓåÆ channel deactivation.  
**Network audio:** Separate dequeue from shared queue; ducking applied to non-network channels while network audio plays.

## External Dependencies
- **Includes:** `SDL_endian.h` (byte swap), `cseries.h` (common types), `network_speaker_sdl.h` (network buffer descriptor), `network_audio_shared.h` (network audio constants), `map.h` (dynamic_world state), `Music.h` (Music singleton), `SoundManager.h` (SoundManager singleton)
- **External symbols:** `local_player_index`, `game_is_networked`, `dynamic_world`, `Music::instance()`, `SoundManager::instance()`, SDL audio functions (SDL_LockAudio, SDL_UnlockAudio, SDL_SwapLE16, SDL_SwapBE16)
- **Networking:** `dequeue_network_speaker_data()`, `release_network_speaker_buffer()`, `is_sound_data_disposable()` (defined elsewhere)
