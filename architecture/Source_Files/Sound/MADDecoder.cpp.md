# Source_Files/Sound/MADDecoder.cpp

## File Purpose
Implements MP3 audio decoding for the Aleph One game engine using libmad. Provides streaming decoding of MPEG audio frames into PCM samples, handling buffering, endianness conversion, and resource lifecycle.

## Core Responsibilities
- Initialize and manage libmad decoder state (stream, frame, synthesis)
- Open MP3 files and validate format (detect channels, sample rate)
- Stream decode MPEG frames into PCM samples on demand
- Buffer input data from file and manage frame boundaries
- Convert fixed-point audio samples to 16-bit integers with endianness handling
- Support stream rewinding and resource cleanup

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `mad_stream` | struct (libmad) | Input bit stream state and buffering |
| `mad_frame` | struct (libmad) | Decoded MPEG frame header and data |
| `mad_synth` | struct (libmad) | PCM synthesis state (decoded samples) |
| `InputBuffer` | uint8 array | Intermediate buffer for file read data (40KB + guard) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `INPUT_BUFFER_SIZE` | const int | class static | File read buffer capacity (5 ├ù 8192 bytes) |

## Key Functions / Methods

### Open
- Signature: `bool Open(FileSpecifier &File)`
- Purpose: Open an MP3 file and initialize decoding; extract audio format metadata.
- Inputs: FileSpecifier reference to the audio file
- Outputs/Return: `true` on successful initialization, `false` on error
- Side effects: Initializes `Stream`, `Frame`, `Synth`; reads first frame; sets `stereo`, `bytes_per_frame`, `rate`, `sample`
- Calls: `File.Open()`, `DecodeFrame()`
- Notes: Returns false if the first frame cannot be decoded; format info inferred from frame header

### Decode
- Signature: `int32 Decode(uint8* buffer, int32 max_length)`
- Purpose: Decode available PCM samples into the output buffer up to a requested byte length.
- Inputs: Output buffer pointer; maximum bytes to decode
- Outputs/Return: Number of bytes written to buffer
- Side effects: Increments `sample` index; calls `DecodeFrame()` when current frame exhausted; updates `Synth.pcm.length` on EOF
- Calls: `MadFixedToSshort()`, `DecodeFrame()`
- Notes: Adjusts max_length downward (ΓêÆ3 stereo, ΓêÆ1 mono) to avoid buffer overrun; handles channel interleaving (L/R for stereo); respects platform endianness via conditional compilation

### DecodeFrame
- Signature: `bool DecodeFrame()`
- Purpose: Decode the next MPEG frame from the input stream; refill and manage input buffer.
- Inputs: None (reads from file via `Stream`)
- Outputs/Return: `true` on successful frame decode, `false` on EOF or unrecoverable error
- Side effects: Refills `InputBuffer` from file if needed; updates `Stream` state; initializes `Synth` PCM samples; sets `file_done` flag on EOF; recursive call on recoverable errors
- Calls: `file.GetPosition()`, `file.GetLength()`, `file.Read()`, `mad_frame_decode()`, `mad_synth_frame()`, self (recursion)
- Notes: Appends `MAD_BUFFER_GUARD` bytes of zeros at EOF to allow libmad to parse final frame; uses `memmove` to preserve unconsumed data; recursively retries on recoverable errors

### Rewind
- Signature: `void Rewind()`
- Purpose: Reset decoder to beginning of file for replay.
- Inputs: None
- Outputs/Return: None
- Side effects: Finishes and reinitializes all libmad structures; resets file position to 0; resets `sample` and `file_done`; pre-decodes first frame
- Calls: `mad_stream_finish()`, `mad_frame_finish()`, `mad_synth_finish()`, init variants, `file.SetPosition()`, `DecodeFrame()`
- Notes: Invokes full lifecycle (finish/init) rather than partial reset

### Close
- Signature: `void Close()`
- Purpose: Clean up libmad resources.
- Inputs: None
- Outputs/Return: None
- Side effects: Calls finish on all three libmad structures
- Calls: `mad_synth_finish()`, `mad_frame_finish()`, `mad_stream_finish()`
- Notes: Does not close the file (caller responsibility)

### MadFixedToSshort (static inline helper)
- Signature: `int16 MadFixedToSshort(mad_fixed_t Fixed)`
- Purpose: Convert libmad's fixed-point sample to signed 16-bit integer with saturation.
- Inputs: Fixed-point sample value
- Outputs/Return: Saturated int16 sample
- Notes: Clamps to `SHRT_MAX` / `ΓêÆSHRT_MAX` on overflow; bit-shifts by `MAD_F_FRACBITS ΓêÆ 15` to extract 16-bit significand

## Control Flow Notes
Fits into the audio system's streaming lifecycle: typically called once per frame during game update to fill audio buffers for playback. Inherits from `StreamDecoder`, suggesting integration with a polymorphic audio system. The repeated `Decode()` calls drain the current frame's PCM samples, triggering `DecodeFrame()` on exhaustionΓÇölazy frame decoding keeps memory footprint low for long streams.

## External Dependencies
- **libmad**: `<mad.h>` ΓÇö MPEG audio decoder library (provides Stream, Frame, Synth structs and encode/decode functions)
- **Game engine**: `FileSpecifier` (file abstraction), `StreamDecoder` (base class), `cseries.h` (common type definitions)
- **Standard library**: `<cstring>` (implicit via `memmove`); `<climits>` (implicit via `SHRT_MAX`)
- **Platform conditionals**: `ALEPHONE_LITTLE_ENDIAN` (endianness), `HAVE_MAD` (availability guard)
