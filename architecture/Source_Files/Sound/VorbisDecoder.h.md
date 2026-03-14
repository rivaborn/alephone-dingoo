# Source_Files/Sound/VorbisDecoder.h

## File Purpose
Defines the `VorbisDecoder` class that handles decoding and streaming of OGG Vorbis audio files. It inherits from `StreamDecoder` and wraps the libvorbis library to provide a unified interface for audio decoding within the game engine's sound system.

## Core Responsibilities
- Open and close OGG Vorbis audio files via `FileSpecifier`
- Decode Vorbis-compressed audio data into raw PCM samples
- Rewind playback position to the beginning of a file
- Query audio format metadata (bit depth, channels, sample rate, endianness, frame size)
- Manage underlying `OggVorbis_File` handle and vorbis callbacks

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `VorbisDecoder` | Class | Concrete decoder for OGG Vorbis streams; inherits `StreamDecoder` |
| `OggVorbis_File` | Typedef (external) | Opaque vorbis file handle from libvorbis |
| `ov_callbacks` | Struct (external) | Callback function pointers for custom I/O (file reading, seeking, etc.) |

## Global / File-Static State
None.

## Key Functions / Methods

### Open
- Signature: `bool Open(FileSpecifier& File)`
- Purpose: Open and initialize a Vorbis file for decoding
- Inputs: File specifier referencing the OGG Vorbis file
- Outputs/Return: `true` if successful, `false` otherwise
- Side effects: Initializes `ov_file`, sets `stereo` and `rate` member variables
- Calls: (implementation not visible; calls vorbis library functions)
- Notes: Conditional on `HAVE_VORBISFILE` preprocessor flag

### Decode
- Signature: `int32 Decode(uint8* buffer, int32 max_length)`
- Purpose: Extract and decode the next chunk of PCM audio data from the stream
- Inputs: Destination buffer pointer, maximum bytes to decode
- Outputs/Return: Number of bytes actually decoded (or Γëñ0 on end-of-stream/error)
- Side effects: Advances playback position in `ov_file`
- Calls: (implementation not visible; calls vorbis decoding functions)
- Notes: Raw PCM output format depends on audio file; format details queried via accessors

### Rewind
- Signature: `void Rewind()`
- Purpose: Seek playback position back to the start of the file
- Side effects: Resets `ov_file` position
- Calls: (implementation not visible)

### Close
- Signature: `void Close()`
- Purpose: Close the vorbis file and clean up resources
- Side effects: Invalidates `ov_file`
- Calls: (implementation not visible)

**Format Query Methods** (trivial accessors):
- `IsSixteenBit()` ΓåÆ always `true` (hardcoded)
- `IsStereo()` ΓåÆ returns `stereo` member
- `IsSigned()` ΓåÆ always `true` (hardcoded)
- `BytesPerFrame()` ΓåÆ returns frame size (2 or 4 bytes depending on mono/stereo)
- `Rate()` ΓåÆ returns `rate` member (sample rate in Hz)
- `IsLittleEndian()` ΓåÆ platform-dependent (macro `ALEPHONE_LITTLE_ENDIAN`)

## Control Flow Notes
This decoder is part of a pluggable decoder abstraction. The game likely maintains a `StreamDecoder` pointer and calls `Open()` ΓåÆ repeated `Decode()` calls ΓåÆ `Rewind()` / `Close()` as part of the audio streaming pipeline. The conditional compilation (`#ifdef HAVE_VORBISFILE`) suggests Vorbis support is optional.

## External Dependencies
- `cseries.h` ΓÇö provides common type definitions (`uint8`, `int32`, `float`) and utilities
- `Decoder.h` ΓÇö defines parent class `StreamDecoder` and `FileSpecifier` (indirectly)
- `<vorbis/vorbisfile.h>` ΓÇö libvorbis public API (conditional)
- `OggVorbis_File`, `ov_callbacks` ΓÇö types from libvorbis (external symbols)
