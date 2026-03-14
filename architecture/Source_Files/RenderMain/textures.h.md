# Source_Files/RenderMain/textures.h

## File Purpose
Low-level bitmap and texture data structure definitions for the Aleph One rendering system. Provides bitmap metadata layout, initialization functions, and color-remapping utilities for managing texture pixel data and rendering state.

## Core Responsibilities
- Define bitmap metadata structure (`bitmap_definition`) for texture representation
- Provide bitmap flags for marking column-order layout, transparency, and MML override state
- Calculate pixel data origin pointers from bitmap headers
- Pre-compute row address arrays for efficient scanline access
- Support color lookup table (CLUT) remapping for palette-based texture operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `bitmap_definition` | struct | Core texture metadata: dimensions, row pitch, flags, bit depth, and array of row pointers to pixel data |
| Bitmap flags enum | enum | Metadata flags: `_COLUMN_ORDER_BIT` (column-major layout), `_TRANSPARENT_BIT` (RLE transparency), `_PATCHED_BIT` (MML override) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SIZEOF_bitmap_definition` | const int | static | Fixed size (30 bytes) of bitmap_definition structure for serialization/validation |

## Key Functions / Methods

### calculate_bitmap_origin
- **Signature:** `pixel8 *calculate_bitmap_origin(struct bitmap_definition *bitmap);`
- **Purpose:** Compute the memory address where pixel data begins, assuming it immediately follows the bitmap_definition structure header in memory.
- **Inputs:** Pointer to bitmap_definition structure.
- **Outputs/Return:** Pointer to first pixel byte.
- **Side effects:** None (read-only computation).
- **Calls:** None visible.
- **Notes:** Relies on assumption that pixel data is packed contiguously after the header; no bounds checking.

### precalculate_bitmap_row_addresses
- **Signature:** `void precalculate_bitmap_row_addresses(struct bitmap_definition *texture);`
- **Purpose:** Populate the `row_addresses` array in bitmap_definition with pointers to each scanline, enabling O(1) row access during rendering.
- **Inputs:** Pointer to bitmap_definition (with `bytes_per_row`, `height`, and `row_addresses[0]` already initialized).
- **Outputs/Return:** None (in-place array population).
- **Side effects:** Writes to `row_addresses` array in the bitmap structure.
- **Calls:** None visible.
- **Notes:** Caller must pre-initialize `bytes_per_row`, `height`, and `row_addresses[0]` before calling.

### remap_bitmap
- **Signature:** `void remap_bitmap(struct bitmap_definition *bitmap, pixel8 *table);`
- **Purpose:** Apply a color lookup table to all pixels in a bitmap (palette remapping).
- **Inputs:** Pointer to bitmap and pointer to 256-byte remap table.
- **Outputs/Return:** None (in-place pixel modification).
- **Side effects:** Modifies all pixel values in the bitmap via lookup.
- **Calls:** Likely calls `map_bytes` internally (not visible in signature).

### map_bytes
- **Signature:** `void map_bytes(byte *buffer, byte *table, int32 size);`
- **Purpose:** Generic byte array remapping utility; applies a 256-entry lookup table to each byte in a buffer.
- **Inputs:** Pointer to byte buffer, lookup table, and buffer size.
- **Outputs/Return:** None (in-place array modification).
- **Side effects:** Modifies buffer contents.
- **Calls:** None visible.
- **Notes:** Low-level primitive used by `remap_bitmap` and similar color operations.

## Control Flow Notes
This file is primarily data-definition and initialization. Functions are called during texture load/setup phases to populate bitmap metadata and apply color palettes. Row pre-calculation happens once per texture to enable fast scanline lookups during rasterization.

## External Dependencies
- **cseries.h** ΓÇö Provides `pixel8`, `byte`, `int16`, `int32` type definitions and platform abstraction macros.
- All function implementations are defined in `textures.c` (not present in this header).
