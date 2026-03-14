# Source_Files/CSeries/cstypes.h

## File Purpose
Platform-independent type definitions and utility macros for the Aleph One game engine. Provides fixed-size integer types, time representation, fixed-point arithmetic support, and cross-platform compatibility shims for Mac/BeOS/SDL builds.

## Core Responsibilities
- Define fixed-size integer types (int8ΓÇôint32, signed and unsigned) across heterogeneous platforms
- Establish TimeType based on platform capabilities (uint32 on Mac, time_t elsewhere)
- Provide fixed-point (16.16) arithmetic types and conversion macros for fractional values
- Define platform-neutral constants (NONE, UNONE, MEG, KILO)
- Supply min/max bounds for integer types where not in standard headers
- Handle OpenGL availability gracefully on systems without native support

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `int8, uint8, int16, uint16, int32, uint32` | typedef | Fixed-size integer types aliased to platform-native equivalents (SInt8/UInt8 on Mac, Sint8/Uint8 on SDL, native on BeOS) |
| `TimeType` | typedef | Platform-dependent time representation (uint32 on Mac Carbon, time_t on BeOS/SDL) |
| `_fixed` | typedef | 16.16 fixed-point signed integer (int32 with implicit 16-bit fractional part) |
| `byte` | typedef | Alias for uint8; legacy notation |
| `GLfloat` | typedef (conditional) | Defined as float if HAVE_OPENGL not set |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `NONE` | enum constant | global | Sentinel value (-1) for "no value" / "invalid" contexts |
| `UNONE` | enum constant | global | Unsigned sentinel (65535 / 0xFFFF) for unsigned contexts |
| `MEG` | const int | global | Binary constant = 0x100000 (1 MiB) |
| `KILO` | const int | global | Binary constant = 0x400L (1 KiB) |

## Key Functions / Methods
None. This is a header-only definitions file; no function implementations.

## Control Flow Notes
Not applicable. This file is purely declarativeΓÇöit establishes types and constants used throughout the engine but contains no executable control flow. Included early in compilation to provide foundation types.

## External Dependencies
- **Standard library**: `<limits.h>` (INT16_MAX, INT16_MIN, INT32_MAX, INT32_MIN fallback definitions)
- **Generated config**: `config.h` (conditional HAVE_OPENGL flag)
- **Platform-specific**:
  - Mac: `<Carbon/Carbon.h>` (if EXPLICIT_CARBON_HEADER set) ΓåÆ SInt8/UInt8/SInt16/UInt16/SInt32/UInt32
  - BeOS: `<support/SupportDefs.h>` ΓåÆ native types + time_t
  - SDL: `<SDL_types.h>`, `<time.h>` ΓåÆ Sint8/Uint8, Sint16/Uint16, Sint32/Uint32 + time_t

---

**Notes**:
- Fixed-point macros (`FIXED_FRACTIONAL_BITS`, `INTEGER_TO_FIXED`, `FIXED_INTEGERAL_PART`, `FIXED_ONE`, `FIXED_ONE_HALF`) assume 16-bit fractional part for 16.16 representation.
- `FOUR_CHARS_TO_INT` constructs a four-character code into a uint32; typical pattern for resource IDs.
- Comment references "IR" (likely intermediate representation) and mentions const performance concern in embedded/console contexts.
- Fallback defines for INT16/INT32 bounds suggest compatibility with older toolchains that lack `<limits.h>`.
