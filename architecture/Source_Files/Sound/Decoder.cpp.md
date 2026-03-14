# Source_Files/Sound/Decoder.cpp

## File Purpose
Implements factory methods for instantiating audio decoders. Attempts to open a file with multiple decoder implementations (based on compilation flags) and returns the first one that succeeds, or null if all fail.

## Core Responsibilities
- Provide static factory method `StreamDecoder::Get()` to create stream decoders for compressed/streaming audio formats
- Provide static factory method `Decoder::Get()` to create decoders for uncompressed/complete-file audio formats
- Try multiple decoder implementations in priority order (SndfileDecoder ΓåÆ BasicIFFDecoder ΓåÆ VorbisDecoder ΓåÆ MADDecoder)
- Manage decoder lifetime using `auto_ptr` to avoid leaks during failed attempts

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### StreamDecoder::Get
- **Signature:** `static StreamDecoder* StreamDecoder::Get(FileSpecifier& File)`
- **Purpose:** Factory method to instantiate and open an appropriate stream decoder for the file
- **Inputs:** `File` ΓÇô FileSpecifier reference to the audio file to decode
- **Outputs/Return:** Pointer to an open StreamDecoder instance; null (0) if no decoder successfully opens the file
- **Side effects:** Allocates decoder objects; calls `Open()` on each decoder (which performs I/O)
- **Calls:** SndfileDecoder/BasicIFFDecoder/VorbisDecoder/MADDecoder constructors and `Open()` methods
- **Notes:**
  - Tries decoders in priority order: SndfileDecoder (if `HAVE_SNDFILE`), else BasicIFFDecoder, then VorbisDecoder (if `HAVE_VORBISFILE`), then MADDecoder (if `HAVE_MAD`)
  - Uses `auto_ptr::release()` to transfer ownership to caller on success
  - Short-circuits and returns immediately upon first successful `Open()`
  - Supports multiple optional format libraries via preprocessor conditionals

### Decoder::Get
- **Signature:** `static Decoder* Decoder::Get(FileSpecifier& File)`
- **Purpose:** Factory method to instantiate and open a decoder for complete (non-streaming) audio files
- **Inputs:** `File` ΓÇô FileSpecifier reference
- **Outputs/Return:** Pointer to an open Decoder instance; null (0) if no decoder succeeds
- **Side effects:** Allocates decoder objects; calls `Open()` on each decoder
- **Calls:** SndfileDecoder/BasicIFFDecoder constructors and `Open()` methods
- **Notes:**
  - Subset of StreamDecoder::Get: only attempts SndfileDecoder and BasicIFFDecoder
  - Does not try VorbisDecoder or MADDecoder (streaming-only decoders that inherit from StreamDecoder, not Decoder)
  - Respects `HAVE_SNDFILE` conditional for priority between SndfileDecoder and BasicIFFDecoder

## Control Flow Notes
These are initialization/asset-loading functions, not part of a frame loop. Called when audio files must be decodedΓÇölikely invoked during sound engine setup or when a new audio resource is requested.

## External Dependencies
- `Decoder.h` ΓÇö base classes (StreamDecoder, Decoder)
- `BasicIFFDecoder.h`, `MADDecoder.h`, `SndfileDecoder.h`, `VorbisDecoder.h` ΓÇö concrete decoder implementations
- `<memory>` ΓÇö `std::auto_ptr`
- FileSpecifier (from FileHandler.h) ΓÇö file reference type
