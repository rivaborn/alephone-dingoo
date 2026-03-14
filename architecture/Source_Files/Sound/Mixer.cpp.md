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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Mixer::Header` | struct | Metadata wrapper (format, pointers, loop info) for a sound sample |
| `Mixer::Channel` | struct | Per-channel playback state: format flags, data pointers, volume, pitch, queue |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Mixer::m_instance` | `Mixer*` | static | Singleton pattern: single global mixer instance |
| `option_nosound` | `bool` | extern global | Runtime flag to disable all audio (e.g., for testing) |

## Key Functions / Methods

### Start
- **Signature:** `void Start(uint16 rate, bool sixteen_bit, bool stereo, int num_channels, int volume, uint16 samples)`
- **Purpose:** Initialize SDL audio subsystem with requested parameters; create and enable playback channels.
- **Inputs:** Sample rate (Hz), bit depth, channel layout, channel count, main volume (0ΓÇômax), audio buffer size.
- **Outputs/Return:** None (modifies internal state; sets error if SDL fails).
- **Side effects:** Opens SDL audio device; allocates channel vector; starts audio callback loop.
- **Calls:** `SDL_OpenAudio()`, `SDL_PauseAudio()`, `alert_user()` on error.
- **Notes:** Falls back gracefully if `option_nosound` is set or SDL initialization fails; obtained format may differ from requested.

### Stop
- **Signature:** `void Stop()`
- **Purpose:** Cleanly shut down audio playback and release resources.
- **Side effects:** Closes SDL audio device; clears channel vector.
- **Calls:** `SDL_CloseAudio()`.

### BufferSound
- **Signature:** `void BufferSound(int channel, const Header& header, _fixed pitch)`
- **Purpose:** Queue or immediately load a sound onto a channel; if channel is busy, queue for later.
- **Inputs:** Channel index, sound header (format + data), pitch multiplier (fixed-point).
- **Side effects:** Modifies channel state under audio lock; may allocate new Header copy for queue.
- **Calls:** `SDL_LockAudio()`, `SDL_UnlockAudio()`, `Channel::LoadSoundHeader()`, `Channel::BufferSoundHeader()`.
- **Notes:** Thread-safe via lock; queueing allows smooth transitions between sounds.

### Callback / MixerCallback
- **Signature:** `static void MixerCallback(void *usr, uint8 *stream, int len)` / `void Callback(uint8 *stream, int len)`
- **Purpose:** SDL audio callback; routes to instance method and selects Mix template based on obtained format.
- **Inputs:** User data (Mixer instance), raw audio buffer pointer, buffer length in bytes.
- **Outputs/Return:** Fills `stream` with mixed audio samples.
- **Side effects:** Reads from all active channels; modifies global audio state (mix accumulator, channel playback positions).
- **Calls:** Instantiates and calls `Mix<>` template with correct type/channel/signedness.
- **Notes:** Called by SDL from audio thread at regular intervals; no explicit lock (atomicity of read/write assumed for format flags).

### Mix (template function)
- **Signature:** `template <class T, bool stereo, bool is_signed> void Mix(T *p, int len)`
- **Purpose:** Core real-time mixing kernel: iterate output samples, read/interpolate from all channels, apply volumes and effects, write output.
- **Inputs:** Output buffer pointer (typed as sample), sample count.
- **Outputs/Return:** Fills output buffer with mixed and clipped samples.
- **Side effects:** Advances playback counter and data pointer for each channel; transitions sounds at loop/end boundaries; calls Music::FillBuffer() and SoundManager callbacks.
- **Calls:** 
  - Per-sample: `SDL_SwapLE16()`, `SDL_SwapBE16()` for endianness
  - At sound end: `Music::instance()->FillBuffer()`, `SoundManager::instance()->IncrementChannelCallbackCount()`, `SoundManager::instance()->GetNetmicVolumeAdjustment()`
  - Network: `dequeue_network_speaker_data()`, `is_sound_data_disposable()`, `release_network_speaker_buffer()`
- **Notes:** 
  - Linear interpolation for fractional sample positions (counter fractional part).
  - Handles loop restart and queued sound loading.
  - Network audio volume attenuation when netmic is active.
  - Main volume and mute-while-transmitting applied at output.
  - Output clipping to int16 range; format conversion for 8-bit output.

### StartMusicChannel
- **Signature:** `void StartMusicChannel(bool sixteen_bit, bool stereo, bool signed_8bit, int bytes_per_frame, _fixed rate, bool little_endian)`
- **Purpose:** Configure and activate the dedicated music channel with given audio format.
- **Inputs:** Audio format parameters (width, layout, endianness), sample rate ratio.
- **Side effects:** Sets up channel at index `sound_channel_count + MUSIC_CHANNEL`; activates it.

### UpdateMusicChannel
- **Signature:** `void UpdateMusicChannel(uint8* data, int len)`
- **Purpose:** Update music buffer pointers as new data arrives.
- **Inputs:** New data pointer and length.
- **Side effects:** Modifies music channel data/length fields (called from Music module to feed streaming data).

### EnsureNetworkAudioPlaying / StopNetworkAudio
- **Signature:** `void EnsureNetworkAudioPlaying()` / `void StopNetworkAudio()`
- **Purpose:** Start or stop network (microphone) audio from the speaker queue.
- **Side effects:** Dequeues network data and activates the network channel, or deactivates and releases buffer.
- **Calls:** `dequeue_network_speaker_data()`, `is_sound_data_disposable()`, `release_network_speaker_buffer()`.
- **Notes:** Wrapped in `#if !defined(DISABLE_NETWORKING)` blocks; locking protects channel activation and buffer pointer updates.

### PlaySoundResource / StopSoundResource
- **Signature:** `void PlaySoundResource(LoadedResource &rsrc)` / `void StopSoundResource()`
- **Purpose:** Load and play a sound from a resource blob (Mac OS resource format); stop playback.
- **Inputs:** Resource memory (PlaySoundResource).
- **Side effects:** Parses resource header, scans sound commands, loads sound on dedicated resource channel.
- **Calls:** `SDL_RWFromMem()`, `SDL_ReadBE16()`, `SDL_ReadBE32()`, `SoundHeader::Load()`, `SDL_RWclose()`.
- **Notes:** Skips format type records and scans for `0x8051` (bufferCmd); supports Mac resource formats 1 and 2.

### Channel::LoadSoundHeader
- **Signature:** `void LoadSoundHeader(const Header& header, _fixed pitch)`
- **Purpose:** Load sound metadata and data pointers into a channel; compute playback rate.
- **Inputs:** Sound header, pitch multiplier.
- **Side effects:** Sets all channel format/data fields; computes pitch-adjusted rate.
- **Notes:** Loop length clamped to 0 if < 4 bytes (prevents malformed loops).

## Control Flow Notes
- **Initialization:** Game calls `Mixer::instance()->Start()` at startup to initialize SDL and allocate channels.
- **Frame loop:** SDL invokes `MixerCallback()` asynchronously at audio buffer intervals (~10ΓÇô20 ms).
- **Mixing:** `Mix()` template iterates over all samples in the output buffer; per-sample, all active channels are read and mixed.
- **Sound transitions:** When a channel's playback counter indicates a sample boundary is crossed, the Mix function checks for loop or queued sound, or deactivates.
- **Shutdown:** Game calls `Stop()` to close audio device.

## External Dependencies
- **SDL audio:** `SDL_OpenAudio()`, `SDL_CloseAudio()`, `SDL_PauseAudio()`, `SDL_LockAudio()`, `SDL_UnlockAudio()`, `SDL_RWops` family, `SDL_ReadBE16()`, `SDL_ReadBE32()`, `SDL_SwapLE16()`, `SDL_SwapBE16()`, `SDL_AudioSpec`.
- **Game modules (defined elsewhere):**
  - `SoundManager::instance()->GetNetmicVolumeAdjustment()`, `IncrementChannelCallbackCount()`, `parameters.mute_while_transmitting`.
  - `Music::instance()->FillBuffer()`, `InterruptFillBuffer()` (macOS variant).
  - `SoundHeader::Load()` (from SoundManager.h).
  - Network audio: `dequeue_network_speaker_data()`, `is_sound_data_disposable()`, `release_network_speaker_buffer()` (from network_speaker_sdl.h).
  - Game state: `dynamic_world->speaking_player_index`, `local_player_index`, `game_is_networked`.
- **Includes:** `Mixer.h` (header), `interface.h` (for `strERRORS`, `badSoundChannels`).
