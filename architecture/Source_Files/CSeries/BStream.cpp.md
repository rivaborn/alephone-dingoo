# Source_Files/CSeries/BStream.cpp

## File Purpose
Implements binary serialization streams for reading and writing typed data with big-endian byte-order conversion. Provides concrete input/output stream classes wrapping `std::streambuf` to replace an older AStream implementation.

## Core Responsibilities
- Implement position tracking and stream size queries for input/output streams
- Read/write raw byte buffers with error checking and exception handling
- Deserialize integer and floating-point types from input streams with endianness conversion
- Serialize integer and floating-point types to output streams with endianness conversion
- Use SDL endianness macros to handle big-endian byte swapping on multi-byte values
- Throw `std::ios_base::failure` exceptions on bounds violations or incomplete reads/writes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `BIStream` | class (abstract) | Base input stream; concrete `BIStreamBE` specializes for big-endian |
| `BIStreamBE` | class | Big-endian input stream with byte-swap operators |
| `BOStream` | class (abstract) | Base output stream; concrete `BOStreamBE` specializes for big-endian |
| `BOStreamBE` | class | Big-endian output stream with byte-swap operators |

## Global / File-Static State
None.

## Key Functions / Methods

### BIStream::tellg
- **Signature:** `std::streampos tellg() const`
- **Purpose:** Query current read position in the stream
- **Inputs:** None
- **Outputs/Return:** Current position as `streampos`
- **Side effects:** None
- **Calls:** `rdbuf()->pubseekoff()`
- **Notes:** Delegates to streambuf's public interface

### BIStream::maxg
- **Signature:** `std::streampos maxg() const`
- **Purpose:** Query total stream size by seeking to end and restoring position
- **Inputs:** None
- **Outputs/Return:** Stream end position as `streampos`
- **Side effects:** Temporarily seeks to end, then restores original position
- **Calls:** `rdbuf()->pubseekoff()`, `rdbuf()->pubseekpos()`

### BIStream::read
- **Signature:** `BIStream& read(char *s, std::streamsize n) throw(failure)`
- **Purpose:** Read exactly `n` bytes into buffer; throw on short read
- **Inputs:** `s` (buffer), `n` (byte count)
- **Outputs/Return:** `*this` (for chaining)
- **Side effects:** Advances stream position; throws on insufficient data
- **Calls:** `rdbuf()->sgetn()`

### BIStream::ignore
- **Signature:** `BIStream& ignore(std::streamsize n) throw(failure)`
- **Purpose:** Skip forward `n` bytes; throw if not enough data available
- **Inputs:** `n` (byte count to skip)
- **Outputs/Return:** `*this`
- **Side effects:** Advances read position; throws on bounds violation
- **Calls:** `rdbuf()->in_avail()`, `rdbuf()->pubseekoff()`
- **Notes:** Typo in error message: "bounc" instead of "bound"

### BIStream::operator>> (uint8, int8)
- **Signature:** `BIStream& operator>>(uint8/int8& value) throw(failure)`
- **Purpose:** Deserialize single byte; convert to signed if needed
- **Inputs:** Reference to value to fill
- **Outputs/Return:** `*this` (for chaining)
- **Side effects:** Advances read position or throws on error
- **Calls:** `read()` (base); indirect recursive call for int8 variant

### BIStreamBE::operator>> (uint16, int16, uint32, int32, double)
- **Signature:** `BIStream& operator>>(uint16/int16/uint32/int32/double& value) throw(failure)`
- **Purpose:** Deserialize multi-byte big-endian value with automatic byte swap
- **Inputs:** Reference to target value
- **Outputs/Return:** `*this`
- **Side effects:** Reads bytes and performs endianness conversion; throws on short read
- **Calls:** `read()`, SDL_SwapBE16/32/64(), `memcpy()` for double
- **Notes:** Double serialized as uint64 bitwise; signed variants convert through unsigned

### BOStream::tellp / maxp
- **Signature:** `std::streampos tellp/maxp() const`
- **Purpose:** Query write position (tellp) or stream size (maxp)
- **Inputs:** None
- **Outputs/Return:** Position as `streampos`
- **Side effects:** maxp temporarily seeks to end and restores position
- **Calls:** `rdbuf()->pubseekoff()`, `rdbuf()->pubseekpos()`

### BOStream::write
- **Signature:** `BOStream& write(const char *s, std::streamsize n) throw(failure)`
- **Purpose:** Write exactly `n` bytes from buffer; throw on short write
- **Inputs:** `s` (buffer), `n` (byte count)
- **Outputs/Return:** `*this`
- **Side effects:** Advances write position; throws on insufficient write
- **Calls:** `rdbuf()->sputn()`

### BOStream::operator<< (uint8, int8) / BOStreamBE::operator<< (uint16ΓÇôdouble)
- **Signature:** `BOStream& operator<<(value) throw(failure)`
- **Purpose:** Serialize typed values with big-endian byte swapping for multi-byte types
- **Inputs:** Value to write
- **Outputs/Return:** `*this`
- **Side effects:** Advances write position; may throw on error
- **Calls:** `write()`, SDL_SwapBE16/32/64(), `memcpy()` for double
- **Notes:** Signed variants cast through unsigned; double treated as bitwise uint64

## Control Flow Notes
This is a utility library with no frame-based flow. Used elsewhere in the engine for binary serialization of game data (configs, saves, network messages, etc.). The BE (big-endian) variants match the game's chosen byte order for persistent formats.

## External Dependencies
- **Standard Library:** `<streambuf>`, `<ios_base>`, `<cstring>` (implicit for memcpy)
- **SDL:** `SDL/SDL_endian.h` for `SDL_SwapBE16()`, `SDL_SwapBE32()`, `SDL_SwapBE64()`
- **Project:** `cseries.h` (defines `uint8`, `int8`, `uint16`, `int16`, `uint32`, `int32`, `Uint64` type aliases)
