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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Music | class | Singleton managing all music playback and playlists |
| StreamDecoder | class pointer | Abstract audio decoder for various file formats |
| FileSpecifier | class | File reference for music file paths |
| GM_Random | struct | Random number generator for playlist shuffling |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| m_instance | static Music* | global | Singleton instance (lazy-initialized) |

## Key Functions / Methods

### instance()
- Signature: `static Music *instance()`
- Purpose: Accessor for the singleton `Music` instance; lazy-initializes on first call
- Inputs: None
- Outputs/Return: Pointer to static `Music` instance
- Side effects: Creates singleton if not already instantiated
- Calls: Constructor (Music()) on first call only
- Notes: Thread-safe pattern relies on single-threaded initialization

### Open(FileSpecifier *file)
- Signature: `void Open(FileSpecifier *file)`
- Purpose: Open and initialize a music file for playback
- Inputs: Pointer to file specifier
- Outputs/Return: None (state changes via Load())
- Side effects: Calls `Load()`, sets `music_file`, initializes `decoder` and audio format fields
- Calls: `Load()`, `StreamDecoder::Get()`
- Notes: Replaces any previously loaded music

### FillBuffer()
- Signature: `bool FillBuffer()`
- Purpose: Decode audio data into the ring buffer for playback; called by audio subsystem
- Inputs: None (uses `decoder` and `music_buffer` state)
- Outputs/Return: Boolean success; updates `music_buffer`
- Side effects: Calls `decoder->Decode()`, handles fade calculations, may trigger level music loading
- Calls: `StreamDecoder::Decode()`, `GetLevelMusic()`, `LoadLevelMusic()`
- Notes: Applies fade-out multiplier during `music_fading` interval; triggers next level song on EOF

### FadeOut(short duration)
- Signature: `void FadeOut(short duration)`
- Purpose: Initiate music fade-out over specified duration (milliseconds)
- Inputs: Duration in ticks/ms
- Outputs/Return: None (sets state flags)
- Side effects: Sets `music_fading=true`, `music_fade_start`, `music_fade_duration`
- Calls: None directly
- Notes: Fade is applied by `FillBuffer()` during decoding

### PreloadLevelMusic() / StopLevelMusic()
- Signature: `void PreloadLevelMusic()` / `void StopLevelMusic()`
- Purpose: Prepare level playlist or stop current level music playback
- Inputs: None
- Outputs/Return: None (modifies playlist and state)
- Side effects: `PreloadLevelMusic()` calls `LoadLevelMusic()` if `IsLevelMusicActive()` true; `StopLevelMusic()` closes decoder and clears state
- Calls: `LoadLevelMusic()`, `Close()`
- Notes: Level music loaded on-demand during `FillBuffer()` when current song ends

### Idle()
- Signature: `void Idle()`
- Purpose: Per-frame update hook for music state (fade progression, buffer fills)
- Inputs: None
- Outputs/Return: None
- Side effects: Updates fade state; may trigger buffer refills
- Calls: (Implementation-dependent; likely `FillBuffer()`)
- Notes: Called regularly by main game loop

### CheckVolume()
- Signature: `void CheckVolume()`
- Purpose: Synchronize internal volume with `SoundManager` settings
- Inputs: None (queries `SoundManager::instance()->parameters.music`)
- Outputs/Return: None
- Side effects: Adjusts playback volume in decoder/mixer
- Calls: `SoundManager::instance()`, `GetVolumeLevel()`
- Notes: Allows dynamic music volume adjustment without restarting playback

## Control Flow Notes
- **Init**: `SetupIntroMusic()` or `Open()` called during engine startup to load intro or first music file
- **Per-frame**: `Idle()` called regularly to update fade state and manage buffer refills
- **Audio callback**: `FillBuffer()` invoked by SDL/audio subsystem when mixing audio; responsible for decoding the next chunk of PCM data
- **Level transitions**: `PreloadLevelMusic()` loads the playlist; as each song ends, `FillBuffer()` calls `GetLevelMusic()` to pick the next track (random or sequential)
- **Shutdown**: `Close()` called to release decoder and clean up resources

## External Dependencies
- `cseries.h`: Platform abstraction macros, SDL, fixed-point types
- `Decoder.h`: `StreamDecoder` abstract class for audio format decoders
- `FileHandler.h`: `FileSpecifier` for file paths
- `Random.h`: `GM_Random` for shuffling level playlists
- `SoundManager.h`: `SoundManager::instance()` for volume parameters
- `<vector>`: STL for `music_buffer` and `playlist`
- SDL (via cseries.h): `SDL_RWops` for file I/O
