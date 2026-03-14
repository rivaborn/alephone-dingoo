# Source_Files/Sound/BasicIFFDecoder.h

## File Purpose
Declares `BasicIFFDecoder`, a concrete decoder for uncompressed AIFF and WAV audio formats. Provides file opening, streaming decode operations, and metadata queries for audio playback in the engine.

## Core Responsibilities
- Opens and parses AIFF/WAV file headers
- Decodes audio frames into client-supplied buffers
- Tracks playback state (position, completion)
- Provides audio format metadata (bit depth, channels, sample rate, endianness)
- Manages file lifecycle and seek operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| BasicIFFDecoder | class | Main decoder; parses and streams uncompressed IFF audio |

## Global / File-Static State
None.

## Key Functions / Methods

### Open
- Signature: `bool Open(FileSpecifier &File)`
- Purpose: Open and validate an AIFF/WAV file, parse headers
- Inputs: File reference
- Outputs/Return: Boolean success flag
- Side effects: Populates format metadata (sixteen_bit, stereo, rate, etc.); opens file handle
- Calls: Not inferable from header

### Decode
- Signature: `int32 Decode(uint8* buffer, int32 max_length)`
- Purpose: Decompress/read audio frames into caller's buffer
- Inputs: Buffer pointer, max bytes to read
- Outputs/Return: Bytes written to buffer (Γëñ max_length)
- Side effects: Advances file position; may update Done() state
- Calls: Not inferable from header

### Rewind / Close / Done
- **Rewind**: Reset playback to start of audio data
- **Close**: Close file handle and clean up
- **Done**: Return whether stream is exhausted
- See notes below

### Property accessors
- `IsSixteenBit()`, `IsStereo()`, `IsSigned()`, `BytesPerFrame()`, `Rate()`, `IsLittleEndian()`, `Frames()`
- Trivial getters; return format metadata set during `Open()`. `Frames()` computes total frame count from `length / bytes_per_frame`.

## Control Flow Notes
Typical usage: `Open()` ΓåÆ repeated `Decode()` calls ΓåÆ `Done()` check ΓåÆ `Close()`. `Rewind()` allows repositioning for repeat playback. Not inferable whether this integrates with a frame/update loop or is event-driven.

## External Dependencies
- **Includes**: `Decoder.h` (base class `StreamDecoder` / `Decoder`)
- **Types used but not defined**: `FileSpecifier`, `OpenedFile` (file I/O abstractions, defined elsewhere)
- **Implicit dependencies**: Audio file I/O, header parsing logic (not visible in header)
