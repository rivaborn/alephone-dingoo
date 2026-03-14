# Source_Files/CSeries/byte_swapping.cpp

## File Purpose
Implements byte-swapping routines for endianness conversion on little-endian systems. Provides in-place swapping of 2-byte and 4-byte values, typically used during data serialization/deserialization (e.g., when reading/writing map files, network data, or other binary formats that may originate from big-endian systems).

## Core Responsibilities
- Swap byte order for 2-byte (16-bit) values in memory
- Swap byte order for 4-byte (32-bit) values in memory
- Support bulk field swapping via loop over multiple contiguous values
- Operate only on little-endian systems (conditional compilation)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `_bs_field` | typedef (short) | Type identifier for swapping operation; uses enum values `_2byte` and `_4byte` |

## Global / File-Static State
None.

## Key Functions / Methods

### byte_swap_memory
- **Signature:** `void byte_swap_memory(void *memory, _bs_field type, int fieldcount)`
- **Purpose:** Swaps byte order in a contiguous block of memory for the specified field type.
- **Inputs:**
  - `memory`: pointer to the memory block to modify
  - `type`: field type (`_2byte` for 16-bit, `_4byte` for 32-bit)
  - `fieldcount`: number of fields to swap (can be negative, per comment)
- **Outputs/Return:** None (void); modifies memory in place.
- **Side effects:** Directly modifies the data at the memory address.
- **Calls:** None visible.
- **Notes:**
  - Only compiled when `ALEPHONE_LITTLE_ENDIAN` is defined.
  - For `_2byte`: swaps bytes at positions 0 and 1.
  - For `_4byte`: swaps bytes 0Γåö3 and 1Γåö2 (converting between big-endian and little-endian 32-bit).
  - Inefficient for large counts; no vectorization or bulk operation optimization.
  - Uses simple byte-swap pattern via temporary variable; no edge-case handling for invalid types.

## Control Flow Notes
Utility function, likely called during frame/update loops or I/O operations when reading/writing binary data from external sources. On big-endian systems, the macro in the header expands to a no-op.

## External Dependencies
- `cseries.h` (umbrella header for core types and SDL integration)
- `byte_swapping.h` (function declaration, enum definitions)
- `uint8` type (from `cstypes.h`, included transitively)
- Endianness detection via `ALEPHONE_LITTLE_ENDIAN` (derived from SDL's `SDL_BYTEORDER`)
