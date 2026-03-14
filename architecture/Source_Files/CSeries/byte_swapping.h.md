# Source_Files/CSeries/byte_swapping.h

## File Purpose
Provides endianness conversion utilities for the Aleph One engine. Declares `byte_swap_memory()` to swap byte order in memory, with conditional compilation to optimize away the function on little-endian systems.

## Core Responsibilities
- Define type field markers for indicating byte counts during swaps
- Declare the primary byte-swapping function
- Conditionally disable swapping on systems that don't need it (little-endian)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `_bs_field` | typedef | Short type used as field-type parameter for `byte_swap_memory()` |
| `_2byte`, `_4byte` | enum | Named constants (`-2`, `-4`) marking 2-byte and 4-byte field sizes |

## Global / File-Static State
None.

## Key Functions / Methods

### byte_swap_memory
- **Signature:** `void byte_swap_memory(void *memory, _bs_field type, int fieldcount)`
- **Purpose:** Swap byte order in a block of memory for a specified data type and field count.
- **Inputs:**
  - `memory`: pointer to memory block to swap
  - `type`: field size marker (`_2byte` or `_4byte`)
  - `fieldcount`: number of fields to swap
- **Outputs/Return:** None (void)
- **Side effects:** Modifies memory in-place; on little-endian systems (when `ALEPHONE_LITTLE_ENDIAN` is defined), becomes a no-op.
- **Calls:** Not inferable from this file.
- **Notes:** Conditionally compiled out entirely on little-endian architectures, suggesting this engine targets big-endian or multi-endian platforms (classic Mac OS, network byte order handling, or binary file I/O across platforms).

## Control Flow Notes
Not inferable from this header. Likely called by file I/O, networking, or data-format parsing modules that need endianness conversion.

## External Dependencies
- `<stddef.h>` (standard size/offset utilities)
