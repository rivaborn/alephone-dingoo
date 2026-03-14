# Source_Files/CSeries/BStream.h

## File Purpose
Provides binary stream serialization/deserialization classes that wrap C++ `streambuf` for reading and writing typed binary data. Intended as a replacement for `AStream`. Supports multiple integer/floating-point types with endianness handling (big-endian variant included).

## Core Responsibilities
- Abstract base for binary input/output operations via `streambuf`
- Define operator overloads for serializing/deserializing primitive types (int8, uint8, int16, uint16, int32, uint32, double)
- Support positional queries (`tellg`/`tellp`, `maxg`/`maxp`) on streams
- Provide concrete big-endian (BE) implementations for platform-specific byte order
- Expose raw byte read/write (`read()`, `write()`, `ignore()`)
- Manage streambuf attachment/detachment

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `basic_bstream` | class | Base wrapper around `std::streambuf`; provides `rdbuf()` accessor |
| `BIStream` | class (abstract) | Input stream base; declares virtual operators for multi-byte types |
| `BIStreamBE` | class | Concrete big-endian input stream; implements all `operator>>` |
| `BOStream` | class (abstract) | Output stream base; declares virtual operators for multi-byte types |
| `BOStreamBE` | class | Concrete big-endian output stream; implements all `operator<<` |

## Global / File-Static State
None.

## Key Functions / Methods

### basic_bstream::basic_bstream
- **Signature:** `basic_bstream(std::streambuf* sb)`
- **Purpose:** Initialize stream with a streambuf pointer
- **Inputs:** `sb` ΓÇö target streambuf
- **Outputs/Return:** None (constructor)
- **Side effects:** Stores pointer

### basic_bstream::rdbuf()
- **Signature:** `std::streambuf* rdbuf() const; std::streambuf* rdbuf(std::streambuf* new_sb)`
- **Purpose:** Get or replace the underlying streambuf
- **Inputs:** `new_sb` (setter variant)
- **Outputs/Return:** Current/old streambuf pointer

### BIStream::operator>> (int16, uint16, int32, uint32, double)
- **Signature:** `virtual BIStream& operator>>(T& value) throw(failure)`
- **Purpose:** Read typed value from stream (abstract in base; implemented in `BIStreamBE`)
- **Inputs:** Reference to value to populate
- **Outputs/Return:** `*this` (for chaining)
- **Side effects:** Advances streambuf read position; throws `std::ios_base::failure` on I/O error

### BIStream::read, ignore
- **Signature:** `BIStream& read(char* s, std::streamsize n); BIStream& ignore(std::streamsize n) throw(failure)`
- **Purpose:** Read/skip raw bytes
- **Inputs:** `s` ΓÇö buffer; `n` ΓÇö byte count
- **Outputs/Return:** `*this`

### BIStream::tellg, maxg
- **Signature:** `std::streampos tellg() const; std::streampos maxg() const`
- **Purpose:** Query read position and stream size
- **Inputs:** None
- **Outputs/Return:** Stream position/size

### BOStream::operator<< (int16, uint16, int32, uint32, double)
- **Signature:** `virtual BOStream& operator<<(T value) throw(failure)`
- **Purpose:** Write typed value to stream (abstract in base; implemented in `BOStreamBE`)
- **Inputs:** Value to write
- **Outputs/Return:** `*this` (for chaining)
- **Side effects:** Advances streambuf write position; throws on I/O error

### BOStream::write
- **Signature:** `BOStream& write(const char* s, std::streamsize n) throw(failure)`
- **Purpose:** Write raw bytes
- **Inputs:** `s` ΓÇö source buffer; `n` ΓÇö byte count
- **Outputs/Return:** `*this`

### BOStream::tellp, maxp
- **Signature:** `std::streampos tellp() const; std::streampos maxp() const`
- **Purpose:** Query write position and stream capacity
- **Inputs:** None
- **Outputs/Return:** Stream position/size

## Control Flow Notes
These classes are utility/library code for game data serialization. Likely used during:
- Game save/load operations
- Network packet assembly/disassembly
- Asset file import/export

The abstract `BIStream`/`BOStream` force endianness specialization; `BIStreamBE`/`BOStreamBE` implement big-endian conversions. Single-byte types (int8, uint8) are implemented in the base classes (no endianness concern).

## External Dependencies
- `cseries.h` ΓÇö defines `uint8`, `int8`, `int16`, `uint16`, `int32`, `uint32` (via `cstypes.h`)
- `<streambuf>` ΓÇö `std::streambuf`, `std::streampos`, `std::ios_base::failure`
- All integer typedefs defined elsewhere (not in this file)
