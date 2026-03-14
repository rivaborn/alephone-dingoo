# Source_Files/Files/Packing.h

## File Purpose
Provides endianness-aware binary serialization/deserialization utilities for Marathon series game data. Converts between packed big-endian stream format (as stored in game files) and native host alignment, with optional little-endian support for toolchain compatibility.

## Core Responsibilities
- Serialize individual numerical values to byte streams (ValueToStream family)
- Deserialize individual numerical values from byte streams (StreamToValue family)
- Serialize/deserialize arrays of numerical values (ListToStream, StreamToList)
- Copy raw byte blocks to/from streams (BytesToStream, StreamToBytes)
- Support compile-time endianness selection (big-endian by default)
- Maintain stream pointer advancement during packing/unpacking operations

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### StreamToValue (extern, 4 overloads)
- **Signature:** `void StreamToValue(uint8* &Stream, T &Value)` where T Γêê {uint16, int16, uint32, int32}
- **Purpose:** Deserialize a single numerical value from a packed byte stream.
- **Inputs:** Stream pointer (reference), destination value reference
- **Outputs/Return:** Populates `Value` with deserialized data; advances `Stream`
- **Side effects:** Modifies stream pointer; memory read from stream
- **Calls:** Implementation in Packing.cpp (moved there 2002-08-27 per comment)
- **Notes:** Actual implementation selects big-endian or little-endian variant at compile time via macro

### ValueToStream (extern, 4 overloads)
- **Signature:** `void ValueToStream(uint8* &Stream, T Value)` where T Γêê {uint16, int16, uint32, int32}
- **Purpose:** Serialize a single numerical value into a packed byte stream.
- **Inputs:** Stream pointer (reference), source value
- **Outputs/Return:** None; advances `Stream` and writes data
- **Side effects:** Modifies stream pointer; memory write to stream
- **Calls:** Implementation in Packing.cpp
- **Notes:** Complements StreamToValue; endianness variant selected at compile time

### StreamToList (template, inline)
- **Signature:** `template<class T> void StreamToList(uint8* &Stream, T* List, size_t Count)`
- **Purpose:** Deserialize an array of numerical values from a stream.
- **Inputs:** Stream pointer, array pointer, element count
- **Outputs/Return:** Populates array; advances stream
- **Side effects:** Modifies stream pointer and array contents
- **Calls:** StreamToValue (once per element)
- **Notes:** Loops over array, calling StreamToValue for each element

### ListToStream (template, inline)
- **Signature:** `template<class T> void ListToStream(uint8* &Stream, T* List, size_t Count)`
- **Purpose:** Serialize an array of numerical values into a stream.
- **Inputs:** Stream pointer, array pointer, element count
- **Outputs/Return:** None; advances stream
- **Side effects:** Modifies stream pointer and target stream
- **Calls:** ValueToStream (once per element)
- **Notes:** Complements StreamToList; inverse operation

### StreamToBytes
- **Signature:** `void StreamToBytes(uint8* &Stream, void* Bytes, size_t Count)`
- **Purpose:** Copy raw bytes from stream into a buffer without endianness conversion.
- **Inputs:** Stream pointer, destination buffer, byte count
- **Outputs/Return:** None; advances stream
- **Side effects:** memcpy from stream to buffer; modifies stream pointer
- **Calls:** memcpy
- **Notes:** Raw byte copy; no interpretation of data

### BytesToStream
- **Signature:** `void BytesToStream(uint8* &Stream, const void* Bytes, size_t Count)`
- **Purpose:** Copy raw bytes from a buffer into a stream without endianness conversion.
- **Inputs:** Stream pointer, source buffer, byte count
- **Outputs/Return:** None; advances stream
- **Side effects:** memcpy from buffer to stream; modifies stream pointer
- **Calls:** memcpy
- **Notes:** Inverse of StreamToBytes; raw byte copy

## Control Flow Notes
Not part of main game loop. Used during asset loading/saving phasesΓÇöspecifically when Marathon game data files (packed big-endian format) are read from disk and unpacked into native host structures. Selection between big-endian and little-endian variants is compile-time via preprocessor conditionals (`PACKED_DATA_IS_BIG_ENDIAN` / `PACKED_DATA_IS_LITTLE_ENDIAN`).

## External Dependencies
- `memcpy()` (C standard library)
- Standard integer types: `uint8`, `uint16`, `int16`, `uint32`, `int32` (defined elsewhere, likely platform headers)
