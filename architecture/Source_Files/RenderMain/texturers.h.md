# Source_Files/RenderMain/texturers.h

## File Purpose
Declares the interface for texture rendering operations across different bit depths (8, 16, 32-bit). Defines data structures for tint/shading tables, coordinate systems for horizontal and vertical polygon scanline rendering, and function prototypes for texture fill variants (normal, transparent, translucent, tinted, randomized).

## Core Responsibilities
- Define tint table structures (color remapping) for 8/16/32-bit pixel formats
- Provide rendering parameters and macros for horizontal/vertical polygon texture rasterization
- Declare texture rendering functions supporting normal, transparent, translucent, tinted, and randomized effects
- Support dual texture sizes: standard (128├ù128) and large (256├ù256) variants
- Define function pointer types for dynamic texturer selection

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `tint_table8` | struct | 8-bit color remapping lookup: `index[PIXEL8_MAXIMUM_COLORS]` |
| `tint_table16` | struct | 16-bit RGB component lookup tables (R/G/B channels, `PIXEL16_MAXIMUM_COMPONENT+1` entries each) |
| `tint_table32` | struct | 32-bit RGB component lookup tables (same structure as 16-bit) |
| `_horizontal_polygon_line_header` | struct | Scanline downshift for coordinate calculation |
| `_horizontal_polygon_line_data` | struct | Texture source coordinates (x/y, dx/dy), shading table pointer, screen bounds (x0, x1) |
| `_vertical_polygon_data` | struct | Polygon metadata: downshift, x0 position, width, padding |
| `_vertical_polygon_line_data` | struct | Vertical scanline: shading table, texture source pointer, y-coordinates (y0, y1), texture deltas |
| `horizontal_texturer` | typedef (fn ptr) | Function pointer signature for horizontal rendering |
| `vertical_texturer` | typedef (fn ptr) | Function pointer signature for vertical rendering |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `number_of_shading_tables` | short | extern | Count of pre-computed shading/tinting lookup tables |
| `shading_table_fractional_bits` | short | extern | Fixed-point precision for shading table indexing |
| `shading_table_size` | short | extern | Size of each shading table (typically `PIXEL8_MAXIMUM_COLORS`) |
| `texture_random_seed` | uint16 | extern | Seed for randomized texture fill effect |

## Key Functions / Methods

Rendered as prototypes only; implementations are elsewhere. Functions follow naming convention:
`_[effect]_[orientation]_polygon_lines[bitdepth]`

**Horizontal variants (basic, transparent, translucent, landscape):**
- `_texture_horizontal_polygon_lines8/16/32`: Standard texture fill
- `_big_texture_horizontal_polygon_lines8/16/32`: Large (256px) texture
- `_transparent_texture_horizontal_polygon_lines8/16/32`: Alpha-blended
- `_big_transparent_texture_horizontal_polygon_lines8/16/32`
- `_translucent_texture_horizontal_polygon_lines16/32`: Semi-transparent fill
- `_transpucent_texture_horizontal_polygon_lines16/32`: Transparent outline + translucent fill
- `_big_translucent_texture_horizontal_polygon_lines16/32`
- `_big_transpucent_texture_horizontal_polygon_lines16/32`
- `_landscape_horizontal_polygon_lines8/16/32`: Specialized landscape rendering

**Vertical variants:**
- `_texture_vertical_polygon_lines8/16/32`, `_big_*`: Standard
- `_transparent_texture_vertical_polygon_lines8/16/32`: Transparent
- `_tint_vertical_polygon_lines8/16/32`: Color-tinted
- `_randomize_vertical_polygon_lines8/16/32`: Noise/pattern effect
- `_translucent_texture_vertical_polygon_lines16/32`, `_transpucent_*`: Translucency variants

**Parameter macros standardize signatures:**
- `HORIZONTAL_TEXTURE_PARAMS`: `(texture bitmap, screen bitmap, polygon line data, y0, line count, extra)`
- `VERTICAL_TEXTURE_PARAMS`: `(screen bitmap, polygon data, extra)`

## Control Flow Notes
Called during the frame's polygon rendering phase. Functions implement scanline fill: iterate scanlines, sample texture coordinates (using fixed-point deltas), apply shading/tinting, and write to framebuffer. Horizontal/vertical paths handle different coordinate systems and texture layouts.

## External Dependencies
- `#include "textures.h"`: Defines `bitmap_definition`, pixel types (`pixel8`, `pixel16`, `pixel32`)
- Undefined macros used: `PIXEL8_MAXIMUM_COLORS`, `PIXEL16_MAXIMUM_COMPONENT`, `PIXEL32_MAXIMUM_COMPONENT`, `TRIG_SHIFT`, `WORLD_FRACTIONAL_BITS`, `FIXED_FRACTIONAL_BITS`, `bit_depth`

## Notes
- "IR change/addition" comments indicate modifications (likely author initials), including support for large (256px) textures and SeeThroughLiquids effects
- "Transpucent" is intentional terminology: transparent pixels with translucent opaque regions
- Macro-based parameter lists reduce repetition across 32+ function variants
