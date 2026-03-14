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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | typedef (external) | Represents file paths for music files |
| `StreamDecoder` | class (external) | Decodes audio files; obtained via factory `StreamDecoder::Get()` |
| `GM_Random` | class (external) | PRNG for random playlist ordering |
| `SoundManager` | singleton (external) | Holds global volume and sound parameters |
| `Mixer` | singleton (external) | Audio mixing and output |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Music::m_instance` | `Music*` | static/singleton | Holds single global Music instance |

Instance maintains state: `music_initialized`, `music_play`, `music_level`, `music_intro`, `music_fading`, `music_prelevel`, decoder ptr, audio buffers, file references, playlist, and fade timing.

## Key Functions / Methods

### Open
- **Signature:** `void Music::Open(FileSpecifier *file)`
- **Purpose:** Opens a music file for playback; rewinds if already open, closes previous before loading new.
- **Inputs:** `FileSpecifier *file` (may be null)
- **Outputs/Return:** None
- **Side effects:** Calls `Close()`, sets `music_initialized`, calls `Load()`, updates `music_file`
- **Calls:** `Close()`, `Load()`, `Rewind()` (conditionally)
- **Notes:** On macOS, resets `macos_read_more` and `macos_file_done` flags.

### Load
- **Signature:** `bool Music::Load(FileSpecifier &song_file)`
- **Purpose:** Creates decoder for file; caches audio format properties (sample rate, channels, bit depth).
- **Inputs:** `FileSpecifier &song_file`
- **Outputs/Return:** `true` if decoder created successfully
- **Side effects:** Deletes old decoder, allocates new `StreamDecoder` via factory; caches format flags (`sixteen_bit`, `stereo`, `signed_8bit`, `bytes_per_frame`, `rate`, `little_endian`)
- **Calls:** `StreamDecoder::Get()`, `Mixer::instance()->obtained.freq`
- **Notes:** Rate is fixed-point scaled relative to mixer output frequency.

### FillBuffer
- **Signature:** `bool Music::FillBuffer()`
- **Purpose:** Decodes one chunk of audio into `music_buffer`; updates mixer on non-macOS, or signals macOS interrupt handler.
- **Inputs:** None
- **Outputs/Return:** `true` if data decoded and sent to mixer
- **Side effects:** Calls `decoder->Decode()`; updates mixer channel or sets macOS flags
- **Calls:** `decoder->Decode()`, `Mixer::instance()->UpdateMusicChannel()` (non-macOS)
- **Notes:** Returns `false` on macOS if `macos_read_more` is false, or if no decoder, or if volume is zero, or if decode fails.

### Idle
- **Signature:** `void Music::Idle()`
- **Purpose:** Per-frame update for music state: restart if stopped, apply fade-out, refill buffer (macOS).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Play()`, `Restart()`, `Pause()`, `CheckVolume()`, `Mixer::instance()->SetMusicChannelVolume()`; updates fade volume via elapsed time
- **Calls:** `Playing()`, `Restart()`, `FillBuffer()` (macOS only), `Mixer::instance()->SetMusicChannelVolume()`
- **Notes:** Fade-out is linear interpolation from max volume to 0 over `music_fade_duration`; pauses when volume reaches 0.

### Play
- **Signature:** `void Music::Play()`
- **Purpose:** Starts music playback by filling buffer and activating mixer's music channel.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `FillBuffer()`, `Mixer::instance()->StartMusicChannel()`, `CheckVolume()`
- **Calls:** `FillBuffer()`, `Mixer::instance()->StartMusicChannel()`, `CheckVolume()`
- **Notes:** No-op if not initialized, SoundManager not initialized, or SoundManager not active.

### Pause
- **Signature:** `void Music::Pause()`
- **Purpose:** Stops music playback immediately and clears fade state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Mixer::instance()->StopMusicChannel()`, clears `music_fading` flag
- **Calls:** `Mixer::instance()->StopMusicChannel()`
- **Notes:** Used by `FadeOut` when fade completes.

### GetLevelMusic
- **Signature:** `FileSpecifier* Music::GetLevelMusic()`
- **Purpose:** Returns next music file from level playlist; advances playlist index sequentially or randomly.
- **Inputs:** None
- **Outputs/Return:** `FileSpecifier*` to next playlist song, or null if playlist empty
- **Side effects:** Increments `song_number` (sequential mode); generates random index via `randomizer.KISS()` (random mode)
- **Calls:** `randomizer.KISS()`
- **Notes:** Clamps `song_number` to valid range; wraps to 0 when exceeded.

### PreloadLevelMusic
- **Signature:** `void Music::PreloadLevelMusic()`
- **Purpose:** Loads level music and flags it for playback after preload delay.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `LoadLevelMusic()`, sets `music_prelevel`, `music_level`, `music_play`
- **Calls:** `LoadLevelMusic()`

**Trivial helpers** (summarized under Notes):
- `SetupIntroMusic()`, `RestartIntroMusic()`: Intro lifecycle
- `Close()`, `Rewind()`: Decoder cleanup / reset
- `FadeOut()`, `Restart()`, `Playing()`, `StopLevelMusic()`, `SeedLevelMusic()`, `CheckVolume()`: State management
- `InterruptFillBuffer()` (macOS only): Interrupt-safe buffer copy for macOS audio callback

## Control Flow Notes

**Initialization:** Lazy singleton; constructor initializes flags and resizes buffers.

**Per-Frame:** `Idle()` is the main update loop, called each frame to:
1. Transition preload state to playing
2. Auto-restart if playback stopped
3. Apply fade-out volume ramping
4. Refill buffer (macOS platform)

**Audio Pipeline:** `Play()` ΓåÆ `FillBuffer()` (decode) ΓåÆ `Mixer::StartMusicChannel()` (activate); thereafter mixer calls `FillBuffer()` or `InterruptFillBuffer()` as needed.

**Shutdown:** `Close()` or `StopLevelMusic()` cleans up decoder and stops mixer channel.

## External Dependencies

- **Mixer.h**: `Mixer::instance()`, `StartMusicChannel()`, `UpdateMusicChannel()`, `StopMusicChannel()`, `SetMusicChannelVolume()`, `MusicPlaying()`
- **SoundManager.h**: `SoundManager::instance()`, `IsInitialized()`, `IsActive()`, `parameters.music` (volume)
- **Decoder.h**: `StreamDecoder::Get()` factory, `Decode()`, `Rewind()`, format queries
- **FileHandler.h**: `FileSpecifier` type
- **Random.h**: `GM_Random::KISS()`, `SetTable()`
- **XML_LevelScript.h**: Included but not directly used in this file
- **SDL**: `SDL_GetTicks()`, `SDL_LockAudio()`, `SDL_UnlockAudio()`
- **cseries.h**: Type definitions (`int32`, `uint32`, `_fixed`)
