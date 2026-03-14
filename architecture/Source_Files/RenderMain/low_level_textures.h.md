# Source_Files/RenderMain/low_level_textures.h

## File Purpose
Header file providing template-based, low-level pixel blending and texture mapping routines for software rasterization. Supports multiple pixel formats (8/16/32-bit), alpha blending modes, transparency checks, tinting, and randomization effects across horizontal and vertical polygon rendering.

## Core Responsibilities
- Pixel averaging and alpha blending for different color depths (ARGB, 565, indexed)
- Horizontal polygon line texture mapping with per-pixel shading lookup
- Landscape-optimized texture mapping (fixed-scale, row-based sampling)
- Vertical polygon line texture mapping with 4-wide batch optimization
- Tinting/color overlay application via lookup tables
- Randomization/dither effects with state-preserved PRNG
- Format-aware operations (extract/blend/compose color channels based on pixel format)
- Transparency and mask-based pixel filtering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `bitmap_definition` | struct (defined elsewhere) | Texture or screen bitmap with row addresses and metadata |
| `view_data` | struct (defined elsewhere) | Camera/viewport state |
| `_horizontal_polygon_line_data` | struct (defined elsewhere) | Per-line source texture coordinates and shading table |
| `_vertical_polygon_data` | struct (defined elsewhere) | Batch vertical polygon metadata (width, x0, downshift) |
| `_vertical_polygon_line_data` | struct (defined elsewhere) | Per-column texture, coordinates, and shading table |
| `tint_table8`, `tint_table16`, `tint_table32` | struct (defined elsewhere) | Color channel lookup tables for tinting |
| `SDL_PixelFormat` | struct (SDL) | Pixel format masks, shifts, and loss for color channels |
| `pixel8`, `pixel16`, `pixel32` | typedef | Pixel data types (8/16/32-bit) |
| `_fixed` | typedef | Fixed-point texture coordinates |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `world_pixels` | `extern SDL_Surface*` | global | Current screen surface; accessed for pixel format during alpha blending |
| `texture_random_seed` | `extern uint16` | global | PRNG seed for randomization effect; updated each frame |
| `number_of_shading_tables` | `extern int` | global | Bounds-check for tint table index (assertion only) |

## Key Functions / Methods

### average(fg, bg)
- **Signature:** `template <typename T> inline T average(T fg, T bg)`
- **Purpose:** Blend two pixels via bitwise average; generic fallback returns foreground unchanged.
- **Inputs:** `fg`, `bg` ΓÇô two pixels of template type
- **Outputs/Return:** Averaged pixel
- **Notes:** Specializations for `pixel32` (ARGB) and `pixel16` (565) use color-format-aware bitwise averaging to avoid overflow.

### alpha_blend(fg, bg, alpha, rmask, gmask, bmask)
- **Signature:** `template <typename T> inline T alpha_blend(T fg, T bg, pixel8 alpha, pixel32 rmask, pixel32 gmask, pixel32 bmask)`
- **Purpose:** Blend foreground pixel over background using per-channel alpha (0ΓÇô255 scale).
- **Inputs:** `fg`, `bg` (pixels), `alpha` (blend factor), `rmask`/`gmask`/`bmask` (color masks)
- **Outputs/Return:** Blended pixel
- **Notes:** Operates independently on R, G, B channels; alpha = 255 yields fg, alpha = 0 yields bg.

### write_pixel(dst, pixel, shading_table, opacity_table, rmask, gmask, bmask)
- **Signature:** `template <typename T, int sw_alpha_blend, bool check_transparency> void inline write_pixel(...)`
- **Purpose:** Write a texture pixel to screen with optional transparency check and blending mode.
- **Inputs:** `dst` (screen ptr), `pixel` (texture index), `shading_table` (lookup), `opacity_table` (per-pixel alpha), color masks
- **Outputs/Return:** Writes to `*dst`; no return
- **Side effects:** Modifies screen memory at `dst`
- **Calls:** `average()`, `alpha_blend()` (depending on `sw_alpha_blend` template arg)
- **Notes:** `check_transparency=true` skips write if pixel==0; `sw_alpha_blend` selects mode: `_sw_alpha_off` (direct), `_sw_alpha_fast` (average), `_sw_alpha_nice` (full alpha blend).

### texture_horizontal_polygon_lines(texture, screen, view, data, y0, x0_table, x1_table, line_count, opacity_table)
- **Signature:** `template <typename T, int sw_alpha_blend> void texture_horizontal_polygon_lines(...)`
- **Purpose:** Rasterize horizontal scanlines of a textured polygon; inner loop samples texture and writes to screen.
- **Inputs:** texture/screen bitmaps, line segment tables, start y, per-line x ranges, line count, optional opacity table
- **Outputs/Return:** Writes directly to screen bitmap
- **Side effects:** Modifies screen memory; reads from SDL surface format (if `_sw_alpha_nice`)
- **Calls:** `write_pixel<T, sw_alpha_blend, false>(...)` per pixel
- **Notes:** Inner loop accumulates `source_x/y` with `source_dx/dy` to map pixels; uses fixed-point shifts `HORIZONTAL_HEIGHT_DOWNSHIFT`, `HORIZONTAL_WIDTH_DOWNSHIFT`.

### landscape_horizontal_polygon_lines(texture, screen, view, data, y0, x0_table, x1_table, line_count)
- **Signature:** `template <typename T> void landscape_horizontal_polygon_lines(...)`
- **Purpose:** Optimized horizontal polygon fill for landscape textures; assumes power-of-2 width, skips y interpolation.
- **Inputs:** texture/screen bitmaps, line segment tables, line count
- **Outputs/Return:** Writes directly to screen bitmap
- **Side effects:** Modifies screen memory
- **Calls:** Inline shading table lookups; no secondary function calls
- **Notes:** Dynamically computes downshift from texture height; simpler than `texture_horizontal_polygon_lines` (no `source_y` interpolation).

### texture_vertical_polygon_lines(screen, view, data, y0_table, y1_table, opacity_table)
- **Signature:** `template <typename T, int sw_alpha_blend, bool check_transparent> void texture_vertical_polygon_lines(...)`
- **Purpose:** Rasterize vertical polygon lines with 4-wide SIMD-style batch optimization for aligned columns.
- **Inputs:** screen bitmap, line data tables, per-column y ranges, optional opacity table
- **Outputs/Return:** Writes directly to screen bitmap
- **Side effects:** Modifies screen memory; reads SDL pixel format (if `_sw_alpha_nice`)
- **Calls:** `write_pixel<T, sw_alpha_blend, check_transparent>(...)`, `copy_check_transparent<T, check_transparent>(...)`
- **Notes:** Three-phase loop: **sync** (leading non-4-aligned columns), **parallel** (4-wide batch, all aligned), **desync** (trailing). Unrolls 4 parallel texture columns to reduce pointer arithmetic overhead. Aborts batch if columns overlap vertically.

### tint_vertical_polygon_lines(screen, view, data, y0_table, y1_table, transfer_data)
- **Signature:** `template <typename T> void tint_vertical_polygon_lines(...)`
- **Purpose:** Apply color tinting to vertical polygon columns; skips transparent (zero) pixels.
- **Inputs:** screen bitmap, line data, y ranges, transfer_data (low byte = tint table index)
- **Outputs/Return:** Writes tinted pixels to screen
- **Side effects:** Modifies screen memory; reads tint table from shading_table and SDL format
- **Calls:** `tint_tables_pointer<T>(...)`, `get_pixel_tint<T>(...)` per pixel
- **Notes:** Texture acts as a mask; only pixels with non-zero texture values are tinted.

### randomize_vertical_polygon_lines(screen, view, data, y0_table, y1_table, transfer_data)
- **Signature:** `template <typename T> void randomize_vertical_polygon_lines(...)`
- **Purpose:** Apply randomization/dither effect to vertical polygon columns; seed persists across calls.
- **Inputs:** screen bitmap, line data, y ranges, transfer_data (drop threshold for PRNG)
- **Outputs/Return:** Writes randomized pixels to screen; updates global `texture_random_seed`
- **Side effects:** Modifies screen memory; updates `texture_random_seed`
- **Calls:** `randomize_vertical_polygon_lines_write<T>(...)` per visible texel
- **Notes:** PRNG is LFSR-based: `(seed >> 1) ^ 0xb400` if LSB set, else `seed >> 1`. Texture acts as mask.

**Trivial helpers** (inline, 1ΓÇô2 lines each):
- `copy_check_transparent(dst, read, shading_table)` ΓÇô conditional pixel copy with transparency check
- `tint_tables_pointer<T>(line, index)` ΓÇô format-aware pointer to tint lookup table (specializations for pixel8/16/32)
- `get_pixel_tint<T>(pixel, tables, fmt)` ΓÇô format-aware tint lookup (specializations for pixel8/16/32; generic returns 0)
- `randomize_vertical_polygon_lines_write<T>(seed)` ΓÇô format-aware conversion of PRNG seed to pixel (specialization for pixel32 expands byte to word)

## Control Flow Notes
This is a header file included into rendering code. Functions are called during the main **frame/rasterization** phase:
- **Before frame:** Pixel format masks are extracted once from `world_pixels` and passed to functions.
- **Per polygon:** A rasterizer dispatches to `texture_horizontal_polygon_lines`, `landscape_horizontal_polygon_lines`, `texture_vertical_polygon_lines`, `tint_vertical_polygon_lines`, or `randomize_vertical_polygon_lines` based on polygon type and rendering mode.
- **Across frame:** Global state (`texture_random_seed`) is read and updated to maintain PRNG state across multiple polygons.

## External Dependencies
- **SDL:** `SDL_Surface`, `SDL_PixelFormat` (accessed via `world_pixels->format`)
- **Structs (defined elsewhere):** `bitmap_definition`, `view_data`, `_horizontal_polygon_line_data`, `_vertical_polygon_data`, `_vertical_polygon_line_data`, `tint_table8`, `tint_table16`, `tint_table32`
- **Macros:** `HORIZONTAL_HEIGHT_DOWNSHIFT`, `HORIZONTAL_WIDTH_DOWNSHIFT`, `LANDSCAPE_WIDTH_BITS`, `LANDSCAPE_TEXTURE_WIDTH_DOWNSHIFT`, `FIXED_INTEGERAL_PART`, `NextLowerExponent` (function), `MAX`, `MIN`
- **Types:** `pixel8`, `pixel16`, `pixel32`, `_fixed`, `uint8`, `uint16`, `uint32`, `byte`
- **Globals:** `texture_random_seed`, `number_of_shading_tables`, `world_pixels`
- **Enums:** `_sw_alpha_off`, `_sw_alpha_fast`, `_sw_alpha_nice` (alpha blend mode selectors)
