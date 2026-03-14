# Source_Files/RenderMain/textures.cpp

## File Purpose
Core texture and bitmap pixel data management for the Aleph One game engine. Handles memory layout calculation, row-address pre-computation for both linear and RLE-compressed bitmaps, and palette-based color remapping operations.

## Core Responsibilities
- Calculate physical memory origin of bitmap pixel data given a `bitmap_definition` structure
- Pre-compute row address pointers for fast per-row pixel access (supports both linear and RLE formats)
- Apply color palette remapping tables to bitmaps (used for dynamic color/lighting effects)
- Support two bitmap encoding schemes: fixed-stride linear and variable-length RLE
- Handle column-order vs. row-order bitmap layouts
- Manage big-endian RLE format in Marathon 2 variant

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `bitmap_definition` | struct (defined in textures.h) | Metadata for a bitmap: dimensions, stride, encoding type, and row address table |

## Global / File-Static State
None.

## Key Functions / Methods

### `calculate_bitmap_origin`
- **Signature:** `pixel8 *calculate_bitmap_origin(struct bitmap_definition *bitmap)`
- **Purpose:** Locate the start of pixel data in memory, accounting for the bitmap_definition structure and row-address table that precede it.
- **Inputs:** Pointer to bitmap_definition
- **Outputs/Return:** Pointer to first pixel byte
- **Side effects:** None (read-only)
- **Calls:** None
- **Notes:** Row-address table size depends on `_COLUMN_ORDER_BIT` flag (width or height entries).

### `precalculate_bitmap_row_addresses`
- **Signature:** `void precalculate_bitmap_row_addresses(struct bitmap_definition *bitmap)`
- **Purpose:** Fill the bitmap's row-address table with pointers to each row's pixel data; enables O(1) row lookup at runtime.
- **Inputs:** Pointer to bitmap_definition (must have `bytes_per_row`, `height`, `row_addresses[0]` initialized)
- **Outputs/Return:** None (modifies bitmap in-place)
- **Side effects:** Writes to `bitmap->row_addresses` array
- **Calls:** None
- **Notes:** Handles two formats: (1) linear bitmaps with fixed `bytes_per_row`, and (2) RLE-compressed when `bytes_per_row == NONE`. RLE format differs between Marathon 1 (short run count) and Marathon 2 (big-endian first/last indices). Comments flag little-endian machine compatibility issues in Marathon 1 variant.

### `remap_bitmap`
- **Signature:** `void remap_bitmap(struct bitmap_definition *bitmap, pixel8 *table)`
- **Purpose:** Apply a color remap table to all pixels in a bitmap (e.g., for palette effects or lighting).
- **Inputs:** Pointer to bitmap_definition; pointer to 256-entry lookup table
- **Outputs/Return:** None (modifies bitmap pixels in-place)
- **Side effects:** Overwrites all pixel values via `map_bytes()`
- **Calls:** `map_bytes()`
- **Notes:** Handles both linear and RLE formats. Like `precalculate_bitmap_row_addresses`, differs between Marathon 1 and 2 RLE encodings; Marathon 2 uses big-endian first/last indices.

### `map_bytes`
- **Signature:** `void map_bytes(register byte *buffer, register byte *table, register int32 size)`
- **Purpose:** Apply a lookup table to a contiguous block of bytes (palette remap primitive).
- **Inputs:** Pointer to byte buffer; pointer to 256-entry lookup table; byte count
- **Outputs/Return:** None (modifies buffer in-place)
- **Side effects:** Overwrites buffer
- **Calls:** None
- **Notes:** Simple tight loop; `register` keyword suggests performance-critical path (pre-modern optimization hint).

## Control Flow Notes
- **Initialization:** `calculate_bitmap_origin()` is called to establish the base pixel pointer.
- **Setup/Preprocessing:** `precalculate_bitmap_row_addresses()` is called once per bitmap to populate the row-address lookup table, enabling fast per-row access in the render loop.
- **Runtime:** `remap_bitmap()` and `map_bytes()` apply palette remappings on demand (e.g., for dynamic lighting, active item highlighting, or fullbright mode).

## External Dependencies
- **Includes:** `cseries.h` (platform abstraction, data types), `textures.h` (bitmap_definition, prototypes)
- **External symbols used:** `pixel8`, `byte`, `int32`, `uint16` (defined in cseries/cstypes.h)
- **Conditional compilation:** `MARATHON1` and `MARATHON2` macros gate different RLE handling; `env68k` and segment pragmas are Metrowerks-specific (legacy 68k support)
