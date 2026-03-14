# Source_Files/CSeries/cspixels.h

## File Purpose
Defines pixel color type aliases and provides macros for converting between RGB color values and packed pixel formats (16-bit and 32-bit). Supports color component extraction from pixel values in both 16-bit and 32-bit formats.

## Core Responsibilities
- Define typed aliases for 8-bit, 16-bit, and 32-bit pixel representations
- Provide RGB-to-pixel conversion macros for 16-bit (5-5-5 RGB) format
- Provide RGB-to-pixel conversion macros for 32-bit (8-8-8 RGB) format
- Provide component extraction macros to read R, G, B values from packed pixels
- Define color-space constants (maximum values, component counts)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `pixel8` | typedef (uint8) | 8-bit indexed color pixel |
| `pixel16` | typedef (uint16) | 16-bit packed RGB pixel (5-5-5 format) |
| `pixel32` | typedef (uint32) | 32-bit packed RGB pixel (8-8-8 format) |

## Global / File-Static State
None.

## Key Functions / Methods
None. This header contains only type definitions and preprocessor macros.

## Macro Reference
**16-bit color (565-like, 5 bits per channel):**
- `RGBCOLOR_TO_PIXEL16(r,g,b)` ΓÇô Converts 16-bit RGB values to packed 16-bit pixel
- `RED16(p)`, `GREEN16(p)`, `BLUE16(p)` ΓÇô Extract 5-bit components from 16-bit pixel

**32-bit color (888, 8 bits per channel):**
- `RGBCOLOR_TO_PIXEL32(r,g,b)` ΓÇô Converts 16-bit RGB values to packed 32-bit pixel
- `RED32(p)`, `GREEN32(p)`, `BLUE32(p)` ΓÇô Extract 8-bit components from 32-bit pixel

**Color constants:**
- `PIXEL8_MAXIMUM_COLORS` (256) ΓÇô Palette size for indexed color
- `PIXEL16_MAXIMUM_COMPONENT` (31) ΓÇô Max value for 5-bit channel
- `PIXEL32_MAXIMUM_COMPONENT` (255) ΓÇô Max value for 8-bit channel
- `NUMBER_OF_COLOR_COMPONENTS` (3) ΓÇô RGB channels

## Control Flow Notes
Not applicable; this is a pure utility header with no executable code.

## External Dependencies
- `#include "cstypes.h"` ΓÇô Provides base type definitions (`uint8`, `uint16`, `uint32`)

---

**Notes:**
- Macro design assumes input values in range 0x0000ΓÇô0xFFFF; outputs depend on format (0x00ΓÇô0x1F for 16-bit, 0x00ΓÇô0xFF for 32-bit).
- 16-bit format uses bit-packing: bits 14ΓÇô10 (red), 9ΓÇô5 (green), 4ΓÇô0 (blue).
- 32-bit format uses byte alignment: bytes 2 (red), 1 (green), 0 (blue).
