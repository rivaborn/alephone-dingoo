# Source_Files/Sound/Decoder.h

## File Purpose
Defines abstract interfaces for audio decoding of music and external sounds. Provides two decoder classes: `StreamDecoder` for streaming arbitrary audio data, and `Decoder` (extended with frame counting) for complete file decoders. Uses factory methods to instantiate appropriate concrete implementations based on file type.

## Core Responsibilities
- Define `StreamDecoder` abstract base class for streaming audio frame-by-frame
- Define `Decoder` abstract class extending `StreamDecoder` with total frame info
- Abstract audio format properties (bit depth, channels, sample rate, endianness, signedness)
- Provide factory methods (`Get()`) to create decoder instances for a given file
- Establish contract for file opening, decoding, rewinding, and closing operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `StreamDecoder` | class (abstract base) | Base interface for streaming audio decoders; defines core decode/format operations |
| `Decoder` | class (abstract, inherits StreamDecoder) | Extended decoder interface adding total frame count query |

## Global / File-Static State
None.

## Key Functions / Methods

### StreamDecoder::Open
- Signature: `virtual bool Open(FileSpecifier &File) = 0`
- Purpose: Opens an audio file for decoding
- Inputs: File specification reference
- Outputs/Return: `true` if file opened successfully, `false` otherwise
- Side effects: None visible (decoder state changes handled by implementation)
- Notes: Pure virtual; must be overridden

### StreamDecoder::Decode
- Signature: `virtual int32 Decode(uint8* buffer, int32 max_length) = 0`
- Purpose: Decodes audio frames into the provided buffer
- Inputs: Buffer pointer, maximum bytes to decode
- Outputs/Return: Number of bytes written to buffer (0 on EOF)
- Side effects: Advances internal read position
- Notes: Pure virtual; caller should loop until return value is 0

### StreamDecoder::Rewind
- Signature: `virtual void Rewind() = 0`
- Purpose: Seeks to beginning of audio stream
- Inputs: None
- Outputs/Return: void
- Side effects: Resets decoder position to start

### StreamDecoder::Close
- Signature: `virtual void Close() = 0`
- Purpose: Closes the decoder and releases resources
- Inputs: None
- Outputs/Return: void
- Side effects: Releases file handle and decoder state

### StreamDecoder::Get (factory)
- Signature: `static StreamDecoder* Get(FileSpecifier &File)`
- Purpose: Factory method to instantiate appropriate concrete decoder for the given file
- Inputs: File specification reference
- Outputs/Return: Pointer to newly allocated `StreamDecoder` implementation, or `nullptr` if file type unsupported
- Side effects: Dynamic allocation of decoder object
- Notes: Caller responsible for deletion

### Decoder::Frames
- Signature: `virtual int32 Frames() = 0`
- Purpose: Returns total number of sample frames in the file
- Inputs: None
- Outputs/Return: Frame count
- Notes: Pure virtual; requires complete file length knowledge (unlike streaming decoder)

### Decoder::Get (factory)
- Signature: `static Decoder* Get(FileSpecifier &File)`
- Purpose: Factory method to instantiate appropriate concrete `Decoder` for the given file
- Inputs: File specification reference
- Outputs/Return: Pointer to newly allocated `Decoder` implementation, or `nullptr` if file type unsupported
- Side effects: Dynamic allocation of decoder object
- Notes: Caller responsible for deletion

## Property Query Methods
`IsSixteenBit()`, `IsStereo()`, `IsSigned()`, `BytesPerFrame()`, `Rate()`, `IsLittleEndian()` ΓÇö all pure virtual query methods returning audio format metadata. Used to understand decoded data layout.

## Control Flow Notes
Typical usage: (1) call `Get()` factory to obtain decoder; (2) call `Open()` with file; (3) query format with property methods; (4) loop calling `Decode()` until it returns 0; (5) call `Close()` or use object destruction. `Rewind()` allows seeking back to start for re-reading.

## External Dependencies
- `cseries.h` ΓÇö Core typedefs (`int32`, `uint8`, `float`)
- `FileHandler.h` ΓÇö `FileSpecifier` class for file references
