# Source_Files/Sound/VorbisDecoder.cpp

## File Purpose
Implements the `VorbisDecoder` class, which decodes Ogg Vorbis audio files using libvorbisfile. Bridges SDL file I/O (`SDL_RWops`) with the Vorbis decoding API to enable streaming audio playback in the Aleph One game engine.

## Core Responsibilities
- Wrap SDL file operations (read, seek, close, tell) as callbacks for libvorbisfile
- Open and validate Ogg Vorbis files, extracting audio format metadata (channels, sample rate)
- Decode compressed Vorbis data into PCM audio buffers on demand
- Manage file position (rewind to start)
- Resource cleanup on decoder destruction

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OggVorbis_File` | struct (libvorbis) | Maintains state of an open Vorbis file during decoding |
| `ov_callbacks` | struct (libvorbis) | I/O callback handlers (read, seek, close, tell) |
| `vorbis_info` | struct (libvorbis) | Audio format info: channels, sample rate, bitrate |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sdl_read_func` | function pointer | file-static | Wraps `SDL_RWread` for libvorbis read callbacks |
| `sdl_seek_func` | function pointer | file-static | Wraps `SDL_RWseek` for libvorbis seek callbacks |
| `sdl_close_func` | function pointer | file-static | Wraps `SDL_RWclose` for libvorbis close callbacks |
| `sdl_tell_func` | function pointer | file-static | Wraps `SDL_RWtell` for libvorbis tell callbacks |

## Key Functions / Methods

### VorbisDecoder (constructor)
- **Signature:** `VorbisDecoder()`
- **Purpose:** Initialize decoder state and register SDL I/O callbacks with libvorbis.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `stereo = false`, `rate = 0`; populates `callbacks` structure with SDL wrapper functions.
- **Calls:** (none visible)
- **Notes:** Callback setup happens here; actual file opening is deferred to `Open()`.

### ~VorbisDecoder (destructor)
- **Signature:** `~VorbisDecoder()`
- **Purpose:** Clean up resources on decoder destruction.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Close()` to release libvorbis state.
- **Calls:** `Close()`
- **Notes:** None

### Open
- **Signature:** `bool VorbisDecoder::Open(FileSpecifier &File)`
- **Purpose:** Open and validate an Ogg Vorbis file, extracting audio format metadata.
- **Inputs:** `File` ΓÇô a `FileSpecifier` referencing the audio file
- **Outputs/Return:** `true` if successfully opened and validated; `false` otherwise
- **Side effects:** Modifies `ov_file`, `stereo`, and `rate` members; takes ownership of SDL_RWops from `File.Open()`.
- **Calls:** `File.Open()`, `ov_test_callbacks()`, `ov_test_open()`, `ov_info()`
- **Notes:** Returns false if vorbis_info is null or callbacks/validation fail; opens are staged (`ov_test_callbacks` + `ov_test_open`).

### Decode
- **Signature:** `int32 VorbisDecoder::Decode(uint8* buffer, int32 max_length)`
- **Purpose:** Decompress and read up to `max_length` bytes of PCM audio into the buffer.
- **Inputs:** `buffer` ΓÇô output PCM buffer; `max_length` ΓÇô max bytes to read
- **Outputs/Return:** Bytes actually written to buffer; Γëñ 0 if EOF or error
- **Side effects:** Advances file position in `ov_file`; respects endianness via `IsLittleEndian()`.
- **Calls:** `ov_read()`, `IsLittleEndian()`
- **Notes:** Loops until buffer is full or EOF; `ov_read` parameters: 2-byte samples, signed, mono/stereo determined by `stereo` member.

### Rewind
- **Signature:** `void VorbisDecoder::Rewind()`
- **Purpose:** Seek back to the start of the audio file.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resets file position in `ov_file`.
- **Calls:** `ov_raw_seek()`
- **Notes:** None

### Close
- **Signature:** `void VorbisDecoder::Close()`
- **Purpose:** Release libvorbis and underlying file resources.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `ov_file`; closes SDL_RWops indirectly via libvorbis callback.
- **Calls:** `ov_clear()`
- **Notes:** Safe to call multiple times.

## Control Flow Notes
This decoder is part of a streaming audio system. It is initialized once per audio file, then `Decode()` is called repeatedly during playback to fill audio buffers. The `Rewind()` method allows looping or restart. Resource cleanup happens on destruction or explicit `Close()`. Fits into the audio loading/playback pipeline of the game engine.

## External Dependencies
- **Vorbis/libvorbisfile:** `ov_test_callbacks`, `ov_test_open`, `ov_info`, `ov_read`, `ov_clear`, `ov_raw_seek`, `OggVorbis_File`, `ov_callbacks`, `vorbis_info`
- **SDL:** `SDL_RWops`, `SDL_RWread`, `SDL_RWseek`, `SDL_RWclose`, `SDL_RWtell`
- **Base class:** `StreamDecoder` (defined elsewhere in header)
- **Utilities:** `FileSpecifier`, `IsLittleEndian()`, type aliases `int32`, `uint8` (from `cseries.h`)
