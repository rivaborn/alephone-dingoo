# Source_Files/Sound/SndfileDecoder.h

## File Purpose
Header declaration for `SndfileDecoder`, a concrete audio decoder that wraps libsndfile. Decodes audio files (WAV, FLAC, etc.) into PCM frames for the game's sound system.

## Core Responsibilities
- Open and manage audio files via libsndfile
- Decode audio frames into a PCM buffer on demand
- Query audio format metadata (channels, bit depth, sample rate, endianness)
- Manage file lifecycle (open, rewind, close)
- Implement the `Decoder` abstract interface for polymorphic audio handling

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SndfileDecoder` | class | Concrete decoder for libsndfile-compatible audio formats |
| `SNDFILE` | opaque pointer (libsndfile) | Handle to an open sound file in memory |
| `SF_INFO` | struct (libsndfile) | Audio metadata container (channels, sample rate, frame count) |

## Global / File-Static State
None.

## Key Functions / Methods

### Open
- Signature: `bool Open(FileSpecifier& File)`
- Purpose: Open and initialize a sound file for decoding
- Inputs: `FileSpecifier&` ΓÇö file to open
- Outputs/Return: `bool` ΓÇö success/failure
- Side effects: Populates `sndfile` handle and `sfinfo` metadata
- Calls: libsndfile (implementation not visible)
- Notes: Returns false if file cannot be opened or format is unsupported

### Decode
- Signature: `int32 Decode(uint8* buffer, int32 max_length)`
- Purpose: Decode audio frames and write PCM data to buffer
- Inputs: `uint8* buffer` ΓÇö output buffer; `int32 max_length` ΓÇö max bytes to write
- Outputs/Return: `int32` ΓÇö bytes actually written (0 at EOF)
- Side effects: Advances internal file read position
- Calls: libsndfile (implementation not visible)
- Notes: Returns in 16-bit signed PCM; caller should align `max_length` to `BytesPerFrame()`

### Rewind
- Signature: `void Rewind()`
- Purpose: Reset file position to beginning for looping or replay
- Inputs: None
- Outputs/Return: None
- Side effects: Resets libsndfile internal seek pointer
- Calls: libsndfile (implementation not visible)
- Notes: No error return; behavior undefined if called on unopened file

### Close
- Signature: `void Close()`
- Purpose: Release file handle and free libsndfile resources
- Inputs: None
- Outputs/Return: None
- Side effects: Closes `sndfile` handle
- Calls: libsndfile (implementation not visible)
- Notes: Safe to call multiple times (implementation-dependent)

### Constructor / Destructor
- `SndfileDecoder()` ΓÇö initializes decoder state (likely zeros `sndfile` pointer)
- `~SndfileDecoder()` ΓÇö virtual destructor; likely calls `Close()`

## Notes
**Query methods** ΓÇö trivial accessors: `IsSixteenBit()` (always true), `IsStereo()` (checks `sfinfo.channels == 2`), `IsSigned()` (always true), `BytesPerFrame()` (2 bytes/sample ├ù channel count), `Rate()` (returns `(float)sfinfo.samplerate`), `IsLittleEndian()` (platform-dependent compile-time constant), `Frames()` (returns `sfinfo.frames` ΓÇö total file length).

## Control Flow Notes
Instantiated by `Decoder::Get()` when the engine detects a libsndfile-compatible audio file. After `Open()`, the sound system calls `Decode()` repeatedly in the audio frame loop to fill playback buffers. `Rewind()` supports looping. `Close()` releases resources at shutdown or when playback ends.

## External Dependencies
- `"Decoder.h"` ΓÇö base class `Decoder` and type `FileSpecifier`
- `"sndfile.h"` ΓÇö libsndfile C library (conditional on `HAVE_SNDFILE` macro)
- Indirectly from `Decoder.h`: `cseries.h` (defines `uint8`, `int32`), `FileHandler.h` (file abstraction)
