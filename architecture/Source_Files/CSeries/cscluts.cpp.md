# Source_Files/CSeries/cscluts.cpp

## File Purpose
Color lookup table (CLUT) management for Aleph One game engine on macOS. Converts between platform-agnostic `color_table` structures and macOS Carbon `CTabHandle` resource objects, while maintaining system color definitions (black, white, and UI colors).

## Core Responsibilities
- Convert `color_table` (engine format) to Macintosh `CTabHandle` (native format)
- Convert Macintosh `CTabHandle` (loaded from resources) to engine `color_table` format
- Initialize and expose system color constants (`rgb_black`, `rgb_white`, `system_colors`)
- Validate color counts (clamp to 0ΓÇô256 range) during conversion
- Allocate and populate Macintosh color specification arrays

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` (from cscluts.h) | struct | 16-bit RGB color (red, green, blue fields) |
| `color_table` (from cscluts.h) | struct | Engine's platform-agnostic color palette (count + 256-entry rgb_color array) |
| `RGBColor` (Carbon) | typedef | macOS RGB color representation |
| `ColorSpec` (Carbon) | struct | Entry in a ColorTable (value index + RGBColor) |
| `ColorTable` (Carbon) | struct | macOS color lookup table (seed, size, ctTable array) |
| `LoadedResource` (from FileHandler.h) | class | Wrapper for loaded macOS resource data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rgb_black` | `RGBColor` | global | Black color constant (0x0000, 0x0000, 0x0000) |
| `rgb_white` | `RGBColor` | global | White color constant (0xFFFF, 0xFFFF, 0xFFFF) |
| `system_colors[NUM_SYSTEM_COLORS]` | `RGBColor[2]` | global | System UI colors (gray at 0x2666, light gray at 0xD999) |

## Key Functions / Methods

### build_macintosh_color_table
- **Signature:** `CTabHandle build_macintosh_color_table(color_table *table)`
- **Purpose:** Convert engine `color_table` to macOS Carbon `CTabHandle` resource object
- **Inputs:** Pointer to `color_table` with rgb_color array and count
- **Outputs/Return:** `CTabHandle` (opaque handle to allocated ColorTable); NULL if allocation fails
- **Side effects:** Allocates memory via `NewHandleClear()` for the CTabHandle; calls `GetCTSeed()`
- **Calls:** `NewHandleClear()` (Carbon), `GetCTSeed()` (Carbon)
- **Notes:** Clamps color count to 0ΓÇô256 range; derives `ctSize = n - 1` per macOS convention; maps `ctSeed` from OS-provided value; loops copy rgb_color fields to ColorSpec entries in ctTable

### build_color_table
- **Signature:** `void build_color_table(color_table *table, LoadedResource& clut_rsrc)`
- **Purpose:** Convert loaded macOS `CTabHandle` resource to engine `color_table` format
- **Inputs:** Pointer to output `color_table`; reference to `LoadedResource` wrapping a `CTabHandle`
- **Outputs/Return:** None (void); populates `table->color_count` and `table->colors[0..n-1]`
- **Side effects:** None (read-only of the loaded resource)
- **Calls:** `clut_rsrc.GetHandle()` (FileHandler.h)
- **Notes:** Clamps `ctSize + 1` to 0ΓÇô256 range; loops copy RGBColor fields from ColorSpec array to rgb_color array; handles macOS convention of `ctSize = count - 1`

## Control Flow Notes
These functions are called during resource loading and initialization phases:
- `build_macintosh_color_table()`: Used when the engine needs to export a color table to macOS (prepare for drawing)
- `build_color_table()`: Used when loading a color palette from a macOS resource file (startup or palette change)
- Global color constants are initialized at module load time

## External Dependencies
- **Carbon.h:** `CTabHandle`, `ColorTable`, `ColorSpec`, `RGBColor`, `NewHandleClear()`, `GetCTSeed()`
- **FileHandler.h:** `LoadedResource` class, resource handle management
- **cscluts.h:** `color_table`, `rgb_color` type definitions; extern declarations for global colors
