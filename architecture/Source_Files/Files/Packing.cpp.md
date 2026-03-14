# Source_Files/Files/Packing.cpp

## File Purpose
Implements byte-stream serialization/deserialization functions for converting 16-bit and 32-bit integers to/from packed byte sequences. Supports both big-endian and little-endian formats to handle the Marathon game data format and cross-platform compatibility.

## Core Responsibilities
- Convert uint16/int16 values to/from big-endian byte streams
- Convert uint32/int32 values to/from big-endian byte streams
- Convert uint16/int16 values to/from little-endian byte streams
- Convert uint32/int32 values to/from little-endian byte streams
- Advance stream pointers during read/write operations
- Provide signed/unsigned type overloads via function overloading

## Key Types / Data Structures
None (typedef'd integers only).

## Global / File-Static State
None.

## Key Functions / Methods

### StreamToValueBE (uint16 overload)
- Signature: `void StreamToValueBE(uint8* &Stream, uint16 &Value)`
- Purpose: Extract a 16-bit unsigned integer from a big-endian byte stream.
- Inputs: `Stream` (pointer-to-pointer, incremented by 2); output reference `Value`
- Outputs/Return: Populates `Value` with the 16-bit integer; advances `Stream` by 2 bytes.
- Side effects: Mutates stream pointer; zero-extends bytes during cast.
- Notes: Byte 0 is MSB, Byte 1 is LSB.

### StreamToValueBE (int16 overload)
- Signature: `void StreamToValueBE(uint8* &Stream, int16 &Value)`
- Purpose: Extract a 16-bit signed integer from a big-endian byte stream.
- Inputs: `Stream` (pointer-to-pointer); output reference `Value`
- Outputs/Return: Populates `Value` as signed int16.
- Side effects: Calls unsigned variant, then reinterprets as signed.
- Calls: `StreamToValueBE(uint8*, uint16&)`

### StreamToValueBE (uint32 overload)
- Signature: `void StreamToValueBE(uint8* &Stream, uint32 &Value)`
- Purpose: Extract a 32-bit unsigned integer from a big-endian byte stream.
- Inputs: `Stream`; output reference `Value`
- Outputs/Return: Populates `Value` with the 32-bit integer; advances `Stream` by 4 bytes.
- Side effects: Mutates stream pointer.
- Notes: Byte 0 is MSB, Byte 3 is LSB.

### StreamToValueBE (int32 overload)
- Signature: `void StreamToValueBE(uint8* &Stream, int32 &Value)`
- Purpose: Extract a 32-bit signed integer from a big-endian byte stream.
- Calls: `StreamToValueBE(uint8*, uint32&)`

### ValueToStreamBE (uint16 overload)
- Signature: `void ValueToStreamBE(uint8* &Stream, uint16 Value)`
- Purpose: Pack a 16-bit unsigned integer into a big-endian byte stream.
- Inputs: `Stream` (pointer-to-pointer); `Value` (by value)
- Outputs/Return: Advances `Stream` by 2 bytes.
- Side effects: Writes MSB first, then LSB.

### ValueToStreamBE (int16 overload)
- Signature: `void ValueToStreamBE(uint8* &Stream, int16 Value)`
- Purpose: Pack a 16-bit signed integer into a big-endian byte stream.
- Calls: `ValueToStreamBE(uint8*, uint16)`

### ValueToStreamBE (uint32 overload)
- Signature: `void ValueToStreamBE(uint8* &Stream, uint32 Value)`
- Purpose: Pack a 32-bit unsigned integer into a big-endian byte stream.
- Inputs: `Stream`; `Value`
- Outputs/Return: Advances `Stream` by 4 bytes.
- Side effects: Writes MSB first.

### ValueToStreamBE (int32 overload)
- Calls: `ValueToStreamBE(uint8*, uint32)`

### StreamToValueLE / ValueToStreamLE
- Notes: 8 parallel functions for little-endian byte order (byte order reversed: LSB first).

## Control Flow Notes
These are utility functions called during game file I/O (map loading, save data, asset unpacking). Not part of frame/render/update loops; used at load/parse time and when serializing state.

## External Dependencies
- `cseries.h` ΓÇö platform abstractions, SDL includes
- `cstypes.h` (included via cseries.h) ΓÇö typedef'd fixed-width integer types (uint8, uint16, uint32, int16, int32)
- `Packing.h` ΓÇö header declares public-facing macro names (`StreamToValue`, `ValueToStream`) that route to BE or LE versions
