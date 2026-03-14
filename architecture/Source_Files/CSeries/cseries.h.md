# Source_Files/CSeries/cseries.h

## File Purpose
Master header for the Aleph One game engine providing cross-platform type definitions, macOS API emulation, and portability utilities. Acts as a central include point that bundles platform abstraction, bit-width-specific integer types, endianness detection, and utility macros for the entire CSeries subsystem.

## Core Responsibilities
- Platform detection and configuration management (version, endianness, compiler)
- Endianness abstraction (ALEPHONE_LITTLE_ENDIAN detection via SDL)
- macOS data type emulation for non-Apple platforms (Rect, RGBColor, OSErr, Str255)
- Compiler-specific workarounds (MSVC namespace handling, MVCPP for-loop bug fix)
- Aggregation point for CSeries utility headers (types, macros, colors, strings, fonts, pixels, dialogs)
- DEBUG mode definition

## Key Types / Data Structures
| Name | Kind (struct/enum/class/typedef) | Purpose |
|------|----------------------------------|---------|
| OSErr | typedef | macOS-style error codes (int on non-Apple) |
| Str255 | typedef | Fixed-size Pascal string (256 bytes) |
| Rect | struct | macOS-compatible rectangle with int16 bounds |
| RGBColor | struct | RGB color triplet (uint16 per component) |
| _fixed | typedef (int32) | 16.16 fixed-point arithmetic type |
| rgb_color | struct | Color component container (from cscluts.h) |
| color_table | struct | Indexed color palette (up to 256 colors) |
| TextSpec | struct | Font descriptor with paths and style flags |
| pixel8/16/32 | typedef | Pixel format aliases (uint8/16/32) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| rgb_black, rgb_white | RGBColor | global | Pre-defined standard colors |
| system_colors[] | RGBColor[] | global | System UI color palette (NUM_SYSTEM_COLORS) |
| temporary[] | char[256] | extern | Scratch buffer for string operations |
| ptemporary | Str255* | macro | Cast alias to temporary as Pascal string |
| VERSION | const char* | global (from config.h) | Engine version string |

## Key Functions / Methods
None defined in this file; it is purely declarative. Function declarations reside in the included headers (csmacros.h provides inline templates; others declare utility functions).

## Control Flow Notes
Initialization-phase header. Included early (typically before any engine code) to establish types, constants, and macros. Not involved in frame/update/render/shutdown cycles. Conditional includes adapt types and definitions based on platform (mac, BeOS, SDL) and compiler (#ifdef handling for MVCPP, GCC, APPLE).

## External Dependencies
- **SDL ecosystem**: SDL.h, SDL_byteorder.h, SDL_types.h, SDL_image.h, SDL_net.h (when configured)
- **Standard C library**: time.h, limits.h, string.h
- **macOS/Carbon**: CoreFoundation/CoreFoundation.h, MacTypes.h, Quickdraw.h (conditional on platform)
- **config.h**: Auto-generated configuration (HAVE_OPENGL, HAVE_SDL_IMAGE, etc.)
- **CSeries subsystem headers**: cstypes.h, csmacros.h, cscluts.h, csstrings.h, csfonts.h, cspixels.h, csalerts.h, csdialogs.h, csmisc.h

**Notable macros from included headers**:
- Fixed-point: FIXED_ONE, FIXED_ONE_HALF, INTEGER_TO_FIXED()
- Bitwise: FLAG(), TEST_FLAG32/16(), SET_FLAG32/16()
- Bounds checking: PIN(), MIN(), MAX(), ABS(), SGN()
- Pixel conversion: RGBCOLOR_TO_PIXEL16/32(), RED16/32(), GREEN16/32(), BLUE16/32()
