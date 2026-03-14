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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| MADDecoder | class | Concrete decoder for MP3 files; inherits from `StreamDecoder` |
| OpenedFile | (typedef, from FileHandler) | Manages file handle lifetime |
| mad_stream, mad_frame, mad_synth | (structs, from libmad) | libmad's internal state for decoding pipeline |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| INPUT_BUFFER_SIZE | const int | file-static | 5 ├ù 8192 bytes; input ring-buffer size for MP3 stream |

## Key Functions / Methods

### Open
- Signature: `bool Open(FileSpecifier &File)`
- Purpose: Open an MP3 file and initialize libmad decoder state
- Inputs: File specifier (file path/reference)
- Outputs: true if successful, false otherwise
- Side effects: Opens file handle, initializes `Stream`, `Frame`, `Synth`

### Decode
- Signature: `int32 Decode(uint8* buffer, int32 max_length)`
- Purpose: Decode MP3 stream into PCM samples
- Inputs: Output buffer pointer, max bytes to write
- Outputs: Number of bytes written (Γëñ max_length); 0 if end-of-file
- Side effects: Reads from file, updates `Stream` and `Synth` state
- Calls: `DecodeFrame()` (private)

### Rewind
- Signature: `void Rewind()`
- Purpose: Reset playback to start of stream
- Outputs: None
- Side effects: Seeks file to beginning, resets libmad structures

### Close
- Signature: `void Close()`
- Purpose: Clean up and close audio stream
- Side effects: Closes file handle

### Property Accessors
- `IsSixteenBit()`, `IsStereo()`, `IsSigned()`, `BytesPerFrame()`, `Rate()`, `IsLittleEndian()`: Inline getters returning cached audio properties and platform endianness
- Notes: `IsLittleEndian()` is conditionally compiled based on `ALEPHONE_LITTLE_ENDIAN` macro

### DecodeFrame
- Signature: `bool DecodeFrame()` (private)
- Purpose: Decode one MP3 frame from input buffer into synthesis output
- Side effects: Updates `Stream`, `Frame`, `Synth` state; may refill input buffer from file

## Control Flow Notes
Typical usage: `Open()` ΓåÆ loop [`Decode()` with rewind as needed] ΓåÆ `Close()`.  
Follows streaming decoder pattern: input buffer refilled on demand, output generated incrementally per `Decode()` call. `file_done` flag (member) tracks EOF state.

## External Dependencies
- **libmad**: `mad_stream`, `mad_frame`, `mad_synth` (when `HAVE_MAD` is defined)
- **Base class**: `StreamDecoder` (Decoder.h)
- **File handling**: `FileSpecifier`, `OpenedFile` (FileHandler.h, via cseries.h)
- **Platform**: `ALEPHONE_LITTLE_ENDIAN` macro for endianness detection
