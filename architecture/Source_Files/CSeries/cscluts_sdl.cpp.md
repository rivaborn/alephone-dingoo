# Source_Files/CSeries/cscluts_sdl.cpp

## File Purpose
Implements Mac CLUT (Color Look-Up Table) resource conversion for SDL. Provides global color constants and converts binary CLUT resource data into internal color_table structures with proper endianness handling.

## Core Responsibilities
- Define global color constants (black, white, system palette colors)
- Convert binary Mac CLUT resources to internal color_table format
- Handle big-endian byte order conversion for cross-platform compatibility
- Manage SDL_RWops streams for reading binary resource data
- Validate and clamp color count boundaries (0ΓÇô256)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| RGBColor | struct | 16-bit RGB color (red, green, blue as uint16) |
| color_table | struct | Target structure for converted CLUT (defined elsewhere) |
| rgb_color | struct | Individual color entry in color_table (defined elsewhere) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| rgb_black | RGBColor | global | Constant black (0x0000, 0x0000, 0x0000) |
| rgb_white | RGBColor | global | Constant white (0xffff, 0xffff, 0xffff) |
| system_colors | RGBColor[NUM_SYSTEM_COLORS] | global | System UI palette (gray tones) |

## Key Functions / Methods

### build_color_table
- **Signature**: `void build_color_table(color_table *table, LoadedResource &clut)`
- **Purpose**: Convert Mac CLUT resource format into internal color_table structure
- **Inputs**: 
  - `table`: Destination color_table (modified in-place)
  - `clut`: LoadedResource wrapping binary CLUT data
- **Outputs/Return**: None (modifies table in-place)
- **Side effects**: Allocates and closes SDL_RWops stream; writes color_count and colors array
- **Calls**: SDL_RWFromMem, SDL_RWseek, SDL_ReadBE16, SDL_RWclose
- **Notes**: 
  - Skips first 6 bytes of resource header
  - Reads color count (+ 1) from bytes 6ΓÇô7, clamps to [0, 256]
  - For each color: skips 2 bytes, reads red/green/blue as big-endian uint16
  - Asserts SDL_RWops creation succeeds

## Control Flow Notes
Utility function for resource loading during engine initialization. Not part of frame loop; called when loading color palettes from game resource files.

## External Dependencies
- **Includes**: `cseries.h` (platform/debug macros), `FileHandler.h` (LoadedResource), `SDL_endian.h` (byte order)
- **SDL symbols used**: SDL_RWops (file stream), SDL_RWFromMem, SDL_RWseek, SDL_ReadBE16, SDL_RWclose
- **Symbols defined elsewhere**: RGBColor, color_table, rgb_color, NUM_SYSTEM_COLORS
