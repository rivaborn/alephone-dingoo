# Source_Files/Files/AStream.h

## File Purpose
Provides templated binary stream classes for serialization and deserialization with explicit endian control. Replaces AlephOne's `Packing.h` with clearer API design, allowing runtime selection of Big Endian vs. Little Endian byte ordering through distinct stream types.

## Core Responsibilities
- Define templated base stream class with bounds checking and state management
- Provide input stream (`AIStream`) hierarchy for deserializing binary data
- Provide output stream (`AOStream`) hierarchy for serializing binary data
- Support runtime endian selection via `AIStreamBE`/`AIStreamLE` and `AOStreamBE`/`AOStreamLE` subclasses
- Manage stream state flags (good, fail, bad) and optional exception throwing on state changes
- Implement operator overloading for stream-style I/O syntax (`>>` and `<<`)
- Handle bulk read/write operations and byte skipping via `read()`, `write()`, `ignore()`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `AStream::basic_astream<T>` | Template class | Base stream class managing buffer, position, and state |
| `AStream::failure` | Exception class | Thrown when stream operations fail (bounds check, state errors) |
| `AStream::iostate` | Enum typedef | State flags: `goodbit`, `badbit`, `failbit`, `_M_aiostate_end` |
| `AIStream` | Class | Abstract base for input streams (deserialization) |
| `AIStreamBE` | Class | Big Endian input stream (16/32-bit word deserialization) |
| `AIStreamLE` | Class | Little Endian input stream (16/32-bit word deserialization) |
| `AOStream` | Class | Abstract base for output streams (serialization) |
| `AOStreamBE` | Class | Big Endian output stream (16/32-bit word serialization) |
| `AOStreamLE` | Class | Little Endian output stream (16/32-bit word serialization) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AStream::_S_badbit` | `static const short` | Static | Bit flag for bad state (0x01) |
| `AStream::_S_failbit` | `static const short` | Static | Bit flag for fail state (0x02) |
| `AStream::badbit` | `static const iostate` | Static | Stream state constant for errors |
| `AStream::failbit` | `static const iostate` | Static | Stream state constant for failures |
| `AStream::goodbit` | `static const iostate` | Static | Stream state constant for healthy state (0) |
| `AStream::_M_aiostate_end` | Enum | Static | End-of-state marker (1 << 16) |

## Key Functions / Methods

### `basic_astream::basic_astream(T*, uint32, uint32)` (Constructor)
- **Signature:** `basic_astream(T* __stream, uint32 __length, uint32 __offset)`
- **Purpose:** Initialize stream with buffer, length, and read/write position offset
- **Inputs:** Pointer to buffer start, buffer length in elements, initial position offset
- **Outputs/Return:** Initialized stream object
- **Side effects:** Sets `_M_state` to `badbit` if `__offset > __length`
- **Calls:** None
- **Notes:** Stores end pointer as `__stream + __length`; position validity checked in constructor

### `basic_astream::bound_check(uint32)`
- **Signature:** `bool bound_check(uint32 __delta) throw(AStream::failure)`
- **Purpose:** Check if advancing stream position by `__delta` elements would exceed buffer
- **Inputs:** Number of elements to advance
- **Outputs/Return:** Boolean (true if safe); may throw `failure` if exceptions enabled
- **Side effects:** Sets `failbit` state if bounds exceeded and exceptions disabled
- **Calls:** None
- **Notes:** Protected method; subclasses use for all I/O operations

### `AIStream::operator>>(T&)` (Input operators)
- **Signature (8-bit):** `AIStream& operator>>(uint8&)` and `operator>>(int8&)`
- **Signature (16-bit):** `virtual AIStream& operator>>(uint16&)` = 0 (abstract in AIStream, implemented in BE/LE)
- **Signature (32-bit):** `virtual AIStream& operator>>(uint32&)` = 0 (abstract in AIStream, implemented in BE/LE)
- **Purpose:** Extract typed values from input stream with automatic endian conversion (BE/LE subclasses)
- **Inputs:** Reference to target variable
- **Outputs/Return:** Reference to stream (for chaining)
- **Side effects:** Advances stream position; calls `bound_check()`; sets state on failure
- **Notes:** 8-bit operators delegate to base; 16/32-bit are pure virtual (endian-specific)

### `AOStream::operator<<(T)` (Output operators)
- **Signature (8-bit):** `AOStream& operator<<(uint8)` and `operator<<(int8)`
- **Signature (16-bit):** `virtual AOStream& operator<<(uint16)` = 0
- **Signature (32-bit):** `virtual AOStream& operator<<(uint32)` = 0
- **Purpose:** Write typed values to output stream with automatic endian conversion (BE/LE subclasses)
- **Inputs:** Value to write
- **Outputs/Return:** Reference to stream (for chaining)
- **Side effects:** Advances stream position; calls `bound_check()`; sets state on failure
- **Notes:** Mirrors input operators; enables fluent syntax via operator chaining

### `AIStream::read(T*, uint32)`
- **Signature:** `AIStream& read(T* __list, uint32 __count) throw(AStream::failure)` (template)
- **Purpose:** Read sequence of typed objects using templated operator extraction
- **Inputs:** Pointer to array, count of elements
- **Outputs/Return:** Reference to stream
- **Side effects:** Calls `operator>>` for each element; updates position
- **Calls:** `operator>>` for type `T` (zero-argument loop)
- **Notes:** Specializations exist for `char*`, `unsigned char*`, `signed char*` (byte-level read without endian conversion)

### `AOStream::write(T*, uint32)`
- **Signature:** `AOStream& write(T* __list, uint32 __count) throw(AStream::failure)` (template)
- **Purpose:** Write sequence of typed objects using templated operator insertion
- **Inputs:** Pointer to array, count of elements
- **Outputs/Return:** Reference to stream
- **Side effects:** Calls `operator<<` for each element; updates position
- **Calls:** `operator<<` for type `T` (zero-argument loop)
- **Notes:** Byte-level specializations for raw data without endian conversion

### State management helpers
- `rdstate()`, `setstate()`, `good()`, `fail()`, `bad()`, `exceptions()`: Inline state query/set methods
- `tellg()` / `tellp()`: Return current position for input/output
- `maxg()` / `maxp()`: Return buffer end position for input/output

## Control Flow Notes
Not inferable from headerΓÇöactual I/O and endian byte swapping implemented in `.cpp` files. Header defines the interface and state management only. Pattern suggests usage in **deserialization** (init/load phase) and **serialization** (save phase) of game state or assets. Abstract 16/32-bit operators force subclasses (BE/LE) to implement architecture-specific byte order handling.

## External Dependencies
- **Standard Library:** `<string>` (for `std::string` in `failure` constructor), `<exception>` (base class `std::exception`)
- **Project headers:** `"cstypes.h"` (defines `uint8`, `int8`, `uint16`, `int16`, `uint32`, `int32`)
- **Defined elsewhere:** Actual implementations of `operator>>` / `operator<<` and `read()` / `write()` for integral types (in `.cpp` file, likely with endian swaps)
