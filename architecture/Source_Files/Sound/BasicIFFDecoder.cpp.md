# Source_Files/Sound/BasicIFFDecoder.cpp

## File Purpose
Implements the `BasicIFFDecoder` class, which parses and decodes uncompressed AIFF and WAV audio files. Handles format detection, header parsing, and provides sequential access to audio frame data with playback position tracking.

## Core Responsibilities
- Detect and validate AIFF and WAV file format headers
- Extract audio metadata (sample rate, channels, bit depth, endianness)
- Locate and track audio data chunk boundaries
- Provide buffered reading of audio frames
- Support seeking and rewinding within audio data
- Manage file lifecycle (open, read, close)

## Key Types / Data Structures
None (class definition in header; uses SDL RWops and FileSpecifier types from external context).

## Global / File-Static State
None.

## Key Functions / Methods

### Open()
- **Signature:** `bool Open(FileSpecifier& File)`
- **Purpose:** Parse audio file header, validate format (AIFF or WAV), and extract playback metadata.
- **Inputs:** FileSpecifier reference to audio file on disk.
- **Outputs/Return:** `true` if format recognized and required chunks found; `false` otherwise.
- **Side effects:** Reads file from disk; sets `sixteen_bit`, `stereo`, `signed_8bit`, `bytes_per_frame`, `rate`, `little_endian`, `length`, `data_offset` members; advances file position through chunks.
- **Calls:** `File.Open()`, `file.GetRWops()`, `SDL_ReadBE32()`, `SDL_ReadLE32()`, `SDL_ReadBE16()`, `SDL_ReadLE16()`, `SDL_RWseek()`, `SDL_RWtell()`.
- **Notes:** 
  - AIFF path: expects COMM (format) and SSND (sound data) chunks in big-endian.
  - WAV path: expects fmt (format) and data chunks in little-endian; validates PCM encoding (code 1).
  - AIFF sample rate detection is hardcoded for 80-bit float patterns (44.1 kHz ΓåÆ `0x400eac44`, 22.05 kHz ΓåÆ `0x400dac44`, else 11.025 kHz).
  - Returns `false` if file magic is neither `FORM`/`AIFF` nor `RIFF`/`WAVE`, or if required chunks are missing.

### Decode()
- **Signature:** `int32 Decode(uint8* buffer, int32 max_length)`
- **Purpose:** Read audio frames from file into caller's buffer, advancing playback position.
- **Inputs:** `buffer` (output buffer), `max_length` (max bytes to copy).
- **Outputs/Return:** Number of bytes successfully read (0 on error or file closed).
- **Side effects:** Advances file read position; does not write beyond `data_offset + length` boundary.
- **Calls:** `file.IsOpen()`, `file.GetPosition()`, `file.Read()`.
- **Notes:** Silently clamps read to remaining audio data; returns 0 if file is closed or read fails.

### Done()
- **Signature:** `bool Done()`
- **Purpose:** Test whether all audio data has been consumed.
- **Inputs:** None.
- **Outputs/Return:** `true` if file is closed or file position ΓëÑ `data_offset + length`.
- **Side effects:** None.
- **Calls:** `file.IsOpen()`, `file.GetPosition()`.
- **Notes:** Returns `true` for closed files as a safe default.

### Rewind()
- **Signature:** `void Rewind()`
- **Purpose:** Reset file position to the start of audio data for replay.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Seeks file to `data_offset`.
- **Calls:** `file.IsOpen()`, `file.SetPosition()`.
- **Notes:** No-op if file is closed.

### Close()
- **Signature:** `void Close()`
- **Purpose:** Close file handle and reset data length.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Resets `length` to 0; closes file if open.
- **Calls:** `file.IsOpen()`, `file.Close()`.
- **Notes:** Does not reset format metadata (rate, stereo, etc.); reuse of object after Close is not supported.

## Control Flow Notes
- **Initialization:** `Open()` must be called first to parse file and configure decoder state.
- **Streaming:** `Decode()` is called repeatedly in a frame/render loop to feed audio data to playback engine; caller checks `Done()` between calls to know when to stop.
- **Seeking:** `Rewind()` can reset playback within the same open file.
- **Shutdown:** `Close()` is called when playback finishes or file is no longer needed.

## External Dependencies
- **SDL:** `SDL_endian.h` ΓÇö endian-aware read macros (`SDL_ReadBE32`, `SDL_ReadLE32`, `SDL_ReadBE16`, `SDL_ReadLE16`) and RWops file I/O (`SDL_RWops`, `SDL_RWseek`, `SDL_RWtell`).
- **Decoder** (base class) ΓÇö defined elsewhere; provides virtual interface.
- **FileSpecifier, OpenedFile** ΓÇö file abstraction types defined elsewhere.
- **Unused includes:** `<vector>` and `AStream.h` are included but not referenced in this file.
