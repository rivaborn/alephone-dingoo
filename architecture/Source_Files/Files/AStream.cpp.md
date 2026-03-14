# Source_Files/Files/AStream.cpp

## File Purpose
Implements binary serialization/deserialization (packing/unpacking) with explicit endianness control. Provides four stream classes for converting primitive types to/from byte buffers while enforcing bounds checking and raising exceptions on overflow.

## Core Responsibilities
- Implement base I/O operators (`operator>>` for input, `operator<<` for output) for single-byte types (uint8, int8, bool)
- Provide Big-Endian (BE) and Little-Endian (LE) multi-byte deserialization/serialization with byte-order conversion
- Implement raw byte read/write and stream-skip operations
- Enforce buffer bounds validation and throw `AStream::failure` on overflow
- Define exception class with message handling for serialization errors
- Support method chaining via operator return-by-reference

## Key Types / Data Structures
None defined in this file; all types are defined in `AStream.h`.

| Name | Kind | Purpose |
|------|------|---------|
| `AIStream` | class | Input (deserialization) stream base |
| `AIStreamBE`, `AIStreamLE` | classes | Big/Little-Endian input variants |
| `AOStream` | class | Output (serialization) stream base |
| `AOStreamBE`, `AOStreamLE` | classes | Big/Little-Endian output variants |
| `AStream::failure` | exception class | Serialization error reporting |

## Global / File-Static State
None.

## Key Functions / Methods

### AIStream::operator>>(uint8 &value)
- **Signature:** `AIStream& operator>>(uint8 &value) throw(AStream::failure)`
- **Purpose:** Deserialize a single byte from the input stream.
- **Inputs:** Reference to uint8 variable.
- **Outputs/Return:** Reference to `*this` (for chaining).
- **Side effects:** Increments internal stream position; may set failbit if bounds check fails.
- **Calls:** `bound_check(1)`
- **Notes:** Core single-byte deserialization; int8 and bool operators delegate to this.

### AOStream::operator<<(uint8 value)
- **Signature:** `AOStream& operator<<(uint8 value) throw(AStream::failure)`
- **Purpose:** Serialize a single byte to the output stream.
- **Inputs:** uint8 value.
- **Outputs/Return:** Reference to `*this` (for chaining).
- **Side effects:** Increments internal stream position; may set failbit if bounds check fails.
- **Calls:** `bound_check(1)`
- **Notes:** Core single-byte serialization; int8 and bool operators delegate to this.

### AIStreamBE::operator>>(uint16 &value)
- **Signature:** `AIStream& operator>>(uint16 &value) throw(AStream::failure)`
- **Purpose:** Deserialize a 16-bit big-endian integer.
- **Inputs:** Reference to uint16 variable.
- **Outputs/Return:** Reference to `*this`.
- **Side effects:** Reads 2 bytes and advances stream position; performs byte-order reversal (MSB first).
- **Calls:** `bound_check(2)`
- **Notes:** Manually reads bytes and reconstructs as `(Byte0 << 8) | Byte1`; int16 variant casts result.

### AIStreamLE::operator>>(uint16 &value)
- **Signature:** `AIStream& operator>>(uint16 &value) throw(AStream::failure)`
- **Purpose:** Deserialize a 16-bit little-endian integer.
- **Inputs:** Reference to uint16 variable.
- **Outputs/Return:** Reference to `*this`.
- **Side effects:** Reads 2 bytes and advances stream position; performs byte-order reversal (LSB first).
- **Calls:** `bound_check(2)`
- **Notes:** Reconstructs as `(Byte1 << 8) | Byte0` (reverse of BE).

### AIStreamBE::operator>>(uint32 &value) / AIStreamLE::operator>>(uint32 &value)
- **Signature:** `AIStream& operator>>(uint32 &value) throw(AStream::failure)`
- **Purpose:** Deserialize a 32-bit integer with appropriate endianness.
- **Inputs:** Reference to uint32 variable.
- **Outputs/Return:** Reference to `*this`.
- **Side effects:** Reads 4 bytes and advances stream position.
- **Calls:** `bound_check(4)`
- **Notes:** BE: `(B0 << 24) | (B1 << 16) | (B2 << 8) | B3`; LE: `(B3 << 24) | (B2 << 16) | (B1 << 8) | B0`.

### AOStreamBE::operator<<(uint16 value) / AOStreamLE::operator<<(uint16 value)
- **Signature:** `AOStream& operator<<(uint16 value) throw(AStream::failure)`
- **Purpose:** Serialize a 16-bit integer with appropriate endianness.
- **Inputs:** uint16 value.
- **Outputs/Return:** Reference to `*this`.
- **Side effects:** Writes 2 bytes and advances stream position.
- **Calls:** `bound_check(2)`
- **Notes:** BE writes high byte first; LE writes low byte first.

### AOStreamBE::operator<<(uint32 value) / AOStreamLE::operator<<(uint32 value)
- **Signature:** `AOStream& operator<<(uint32 value) throw(AStream::failure)`
- **Purpose:** Serialize a 32-bit integer with appropriate endianness.
- **Inputs:** uint32 value.
- **Outputs/Return:** Reference to `*this`.
- **Side effects:** Writes 4 bytes and advances stream position.
- **Calls:** `bound_check(4)`
- **Notes:** BE: byte order 24ΓåÆ16ΓåÆ8ΓåÆ0; LE: byte order 0ΓåÆ8ΓåÆ16ΓåÆ24.

### AIStream::read(char *ptr, uint32 count)
- **Signature:** `AIStream& read(char *ptr, uint32 count) throw(AStream::failure)`
- **Purpose:** Read raw bytes into buffer.
- **Inputs:** Destination pointer, byte count.
- **Outputs/Return:** Reference to `*this`.
- **Side effects:** Uses `memcpy` to copy data; advances stream position.
- **Calls:** `bound_check(count)`

### AOStream::write(char *ptr, uint32 count)
- **Signature:** `AOStream& write(char *ptr, uint32 count) throw(AStream::failure)`
- **Purpose:** Write raw bytes to stream.
- **Inputs:** Source pointer, byte count.
- **Outputs/Return:** Reference to `*this`.
- **Side effects:** Uses `memcpy` to copy data; advances stream position.
- **Calls:** `bound_check(count)`

### AStream::basic_astream<T>::bound_check(uint32 delta)
- **Signature:** `bool bound_check(uint32 __delta) throw(AStream::failure)`
- **Purpose:** Validate that reading/writing `delta` bytes will not exceed stream bounds.
- **Inputs:** Number of bytes to validate.
- **Outputs/Return:** `true` if within bounds; `false` if exceeded.
- **Side effects:** Sets `failbit` if bounds exceeded; throws `AStream::failure` exception if exceptions enabled.
- **Calls:** `this->setstate()`, `this->exceptions()`, throws `AStream::failure`
- **Notes:** Returns `false` (not throwing) if exceptions are not enabled; checked before every I/O operation.

### AStream::failure constructors / destructor / what()
- **Signature:** `failure(const std::string& __str) throw()`, `failure(const failure &f)`, `~failure() throw()`, `const char* what() const throw()`
- **Purpose:** Exception class for serialization errors.
- **Inputs/Outputs:** String message (constructor); copied message (copy constructor); null-terminated string (what()).
- **Side effects:** Allocates/frees dynamic string via `strdup`/`free`.
- **Notes:** Manages lifetime of error message; copy constructor duplicates string to avoid dangling pointers.

## Control Flow Notes
Not frame-based. This file is a serialization utility used during network I/O or save/load operations, not during per-frame updates. Initialization occurs via constructors (which validate initial stream position via `bound_check`). No shutdown phase. Error handling via exception throwing in `bound_check` when overflow detected.

## External Dependencies
- `AStream.h` ΓÇö class definitions, type aliases, exception spec
- `<string.h>` ΓÇö `memcpy()` for raw byte copying
- `<string>`, `<exception>` (via `AStream.h`) ΓÇö standard C++ exception base
- `cstypes.h` (via `AStream.h`) ΓÇö integer type definitions (uint8, int8, uint16, int16, uint32, int32)
