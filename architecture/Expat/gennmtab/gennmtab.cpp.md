# Expat/gennmtab/gennmtab.cpp

## File Purpose
Code generation utility for the Expat XML parser. Generates optimized C bitmap lookup tables for validating XML name characters according to the W3C XML specification. This is a build-time tool, not part of the runtime library.

## Core Responsibilities
- Define Unicode character ranges for valid XML name-start characters (letters, ideographs, etc.)
- Define Unicode character ranges for valid name continuation characters (combining marks, digits, extenders, etc.)
- Populate in-memory lookup tables from character range definitions
- Generate compressed bitmap and page-index tables as C code output
- Optimize table representation by detecting duplicate pages and homogeneous regions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `struct range` | struct | Encodes a Unicode character range with `start` and optional `end` fields |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `nmstrt[]` | `struct range[]` | static | XML BaseChar, Ideographic, and related starting characters |
| `name[]` | `struct range[]` | static | Combining marks, digits, extenders, and other name-continuation characters |

## Key Functions / Methods

### setTab
- **Signature:** `void setTab(char *tab, struct range *ranges, size_t nRanges)`
- **Purpose:** Populate a byte lookup table (0=invalid, 1=valid) for all characters in the given ranges.
- **Inputs:** 
  - `tab`: 65536-byte lookup table (one byte per Unicode BMP character)
  - `ranges`: Array of character range definitions
  - `nRanges`: Number of ranges
- **Outputs/Return:** None (modifies `tab` in-place)
- **Side effects:** Writes to the `tab` array; no I/O or allocation.
- **Calls:** (none visible in file)
- **Notes:** Handles both single-character ranges (end=0) and multi-character ranges. Does not validate input.

### printTabs
- **Signature:** `void printTabs(char *tab)`
- **Purpose:** Generate and print optimized C code representing the lookup table. Compresses data by detecting duplicate 256-byte pages and homogeneous 256-byte blocks.
- **Inputs:** `tab`: 131072-byte lookup table (two 65536-byte halves: one for name-start, one for name-continuation)
- **Outputs/Return:** None (writes to stdout as C code)
- **Side effects:** Prints formatted C source code for `namingBitmap[]`, `nmstrtPages[]`, and `namePages[]` arrays.
- **Calls:** `printf`, `putchar`, `memcmp`
- **Notes:** 
  - Processes 512 pages of 256 bytes each
  - Encodes each 256-byte page as 8 x 32-bit bitmaps (256 bits total)
  - Reuses page indices for duplicates (deduplication)
  - Switches output context at page 256 boundary (name-start ΓåÆ name-continuation)

### main
- **Signature:** `int main()`
- **Purpose:** Entry point for the code generation tool. Orchestrates table population and output.
- **Inputs:** None (hardcoded data)
- **Outputs/Return:** 0 on success
- **Side effects:** Allocates 131 KB stack buffer, generates C code to stdout.
- **Calls:** `memset`, `setTab`, `memcpy`, `printTabs`
- **Notes:** 
  - Creates two lookup tables: first for name-start characters, second for name-continuation
  - Copies name-start table, then overlays name-continuation ranges
  - Assumes tab[] has capacity for 2├ù65536 bytes

## Control Flow Notes
This is a code-generation utility, not runtime code. Execution flow:
1. `main()` initializes a dual 65536-byte lookup table
2. `setTab()` marks valid name-start characters in first half
3. First half is copied to second half, then name-continuation ranges are added
4. `printTabs()` reads the dual table and emits optimized C code (bitmap arrays + page indices)
5. The generated code is intended to be compiled into the Expat XML parser for fast character validation

## External Dependencies
- `<string.h>` ΓÇô `memset`, `memcpy`, `memcmp`
- `<stdio.h>` ΓÇô `printf`, `putchar`
- `<stddef.h>` ΓÇô `size_t` type definition

**Defined elsewhere:** XML character classification rules derived from W3C XML 1.0 specification (BaseChar, Ideographic, CombiningChar, Digit, Extender categories).
