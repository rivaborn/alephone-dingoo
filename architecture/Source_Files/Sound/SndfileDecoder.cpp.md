# Source_Files/Sound/SndfileDecoder.cpp

## File Purpose
Implements a sound file decoder using the libsndfile library. Provides functionality to open audio files, decode PCM samples into buffers, seek/rewind playback, and query audio metadata (sample rate, channels, frame count). Conditional compilation (`#ifdef HAVE_SNDFILE`) makes libsndfile support optional.

## Core Responsibilities
- Open sound files via libsndfile and initialize metadata
- Decode audio samples from file into a provided byte buffer
- Seek and rewind file playback position
- Close files and release libsndfile handles
- Expose audio properties (bit depth, stereo/mono, endianness, sample rate, frame count)
- Manage resource cleanup on destruction

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SNDFILE` | opaque pointer (libsndfile) | File handle for libsndfile operations |
| `SF_INFO` | struct (libsndfile) | Audio metadata (sample rate, channels, frames, format) |

## Global / File-Static State
None.

## Key Functions / Methods

### Open
- **Signature:** `bool SndfileDecoder::Open(FileSpecifier& File)`
- **Purpose:** Opens a sound file for decoding
- **Inputs:** `File` ΓÇö FileSpecifier reference with path to audio file
- **Outputs/Return:** `bool` ΓÇö true if file opened successfully, false otherwise
- **Side effects:** Closes any previously open file; initializes `sndfile` handle and `sfinfo` metadata
- **Calls:** `sf_close()`, `sf_open()` (libsndfile)
- **Notes:** Implicitly returns false if `sf_open()` fails (returns null pointer)

### Decode
- **Signature:** `int32 SndfileDecoder::Decode(uint8* buffer, int32 max_length)`
- **Purpose:** Decodes audio samples from the file into the provided buffer
- **Inputs:** `buffer` ΓÇö byte buffer; `max_length` ΓÇö buffer size in bytes
- **Outputs/Return:** Number of bytes successfully decoded and written to buffer
- **Side effects:** Advances file read position
- **Calls:** `sf_read_short()` (libsndfile)
- **Notes:** Divides input byte length by 2 (16-bit samples); multiplies output frame count by 2 to convert back to bytes

### Rewind
- **Signature:** `void SndfileDecoder::Rewind()`
- **Purpose:** Seeks file position back to the start
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Resets file read position to frame 0
- **Calls:** `sf_seek()` (libsndfile)
- **Notes:** Early return if no file is open

### Close
- **Signature:** `void SndfileDecoder::Close()`
- **Purpose:** Closes the audio file and releases the libsndfile handle
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Deallocates libsndfile resources; nullifies `sndfile` pointer
- **Calls:** `sf_close()` (libsndfile)
- **Notes:** Null-safe (checks before closing)

## Control Flow Notes
Typical usage: `Open()` ΓåÆ repeated `Decode()` calls during playback (with optional `Rewind()` to loop) ΓåÆ `Close()` on shutdown or resource release. Destructor calls `Close()` automatically for safety. This decoder integrates into a larger sound subsystem via the `Decoder` base class.

## External Dependencies
- **Inherits from:** `Decoder` (defined elsewhere; abstract sound decoder interface)
- **Uses:** `FileSpecifier` class (file path abstraction, defined elsewhere)
- **Libsndfile API:** `sf_open()`, `sf_close()`, `sf_read_short()`, `sf_seek()`, `SNDFILE`, `SF_INFO`
- **Custom typedefs:** `uint8`, `int32` (assumed defined in project codebase)
- **Conditional compilation:** Entire implementation guarded by `#ifdef HAVE_SNDFILE`
