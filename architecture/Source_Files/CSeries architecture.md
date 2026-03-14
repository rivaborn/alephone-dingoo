# Subsystem Overview

## Purpose
CSeries is a foundational cross-platform compatibility and utilities subsystem that abstracts platform-specific APIs (Mac Carbon, SDL, BeOS) and provides core services including type definitions, string handling, dialogs, graphics management, binary serialization, timing, and error handling. It serves as the substrate upon which higher-level game engine subsystems depend.

## Key Files

| File | Role |
|------|------|
| cseries.h | Master header aggregating type definitions, platform detection, and CSeries utilities |
| cstypes.h | Fixed-size integer types, fixed-point arithmetic, platform constants |
| csmacros.h | Utility macros (MIN, MAX, PIN, bit operations, array bounds checking) |
| csstrings.h/.cpp | String retrieval, PascalΓåöC conversion, encoding (Mac RomanΓåöUnicodeΓåöUTF-8) |
| csalerts.h/.cpp | User dialogs, error/warning logging, assertion handling, fatal error shutdown |
| csdialogs.h/.cpp | Dialog management abstraction (Mac Carbon native and SDL emulation) |
| cscluts.h/.cpp | Color lookup table (CLUT) structures and Mac resource conversion |
| csfonts.h/.cpp | Font specifications and Mac resource loading |
| csmisc.h/.cpp | System timing (tick counter), input polling, platform stubs |
| cspixels.h | Pixel format macros (RGBΓåö16/32-bit packed conversion) |
| cskeys.h | Keyboard scancode constants |
| BStream.h/.cpp | Binary stream serialization/deserialization with endianness handling |
| byte_swapping.h/.cpp | Big-endian byte swapping utilities |
| timer.h/.cpp | Mac Time Manager integration for global millisecond counter |
| mytm.h/.cpp | Task/timer scheduling (Carbon Events, SDL threads, classic Mac TMTask) |
| gdspec.h/.cpp | Graphics device enumeration, resolution/depth configuration, display selection UI |
| csfiles.h/.cpp | File specification retrieval (Mac Classic/Carbon, BeOS) |
| my32bqd.h/.cpp | QuickDraw GWorld abstraction for offscreen rendering |
| NibsUiHelpers.h | RAII wrappers for Mac Carbon/Cocoa NIB resources and event handling |
| readout_data.h | Global performance profiling counters (timing, rendering stats) |
| snprintf.h/.cpp | Portable snprintf/vsnprintf implementations |

## Core Responsibilities

- **Type abstraction:** Define fixed-size integer types (`uint8`ΓÇô`uint32`, `int8`ΓÇô`int32`), fixed-point arithmetic (16.16), and platform-neutral constants across Mac/BeOS/SDL targets
- **Platform detection and abstraction:** Conditional compilation for endianness (big-endian vs. little-endian), graphics APIs (Carbon Quickdraw vs. SDL), and system timing (TMTask vs. Carbon Events vs. SDL timers)
- **String and encoding services:** Resource-based string retrieval (Pascal/C), bidirectional encoding converters (Mac Roman Γåö Unicode Γåö UTF-8), sprintf-like formatting
- **Dialog and UI management:** Abstract dialog creation, control manipulation, event handling, with implementations for both Mac Carbon native dialogs and SDL widget emulation
- **Binary data serialization:** Read/write typed data (integers, floats) with automatic endianness (big-endian) conversion for cross-platform file compatibility
- **Graphics services:** Device enumeration, resolution/color-depth queries and configuration, pixel format conversions (RGB Γåö 16-bit/32-bit packed), color lookup table management
- **System-level utilities:** Timing (tick counter, timer callbacks), input polling (keyboard/mouse), file specification queries, font/resource loading, error/alert dialogs
- **Performance instrumentation:** Global counters for frame timing, rendering statistics, and texture management

## Key Interfaces & Data Flow

**Exposed to other subsystems:**
- **Foundational types** (cstypes.h): `uint8`, `int16`, `uint32`, `TimeType`, fixed-point types, platform constants
- **String/encoding** (csstrings.h): Resource-based C/Pascal strings, encoding conversions, printf-like formatting
- **Dialogs/alerts** (csdialogs.h, csalerts.h): Modal dialogs, control manipulation, user alerts, assertion/error logging
- **Graphics** (gdspec.h, cscluts.h, cspixels.h): Device enumeration/selection, color tables, pixel format macros
- **Serialization** (BStream.h): Binary streams with automatic endianness handling
- **Timing** (timer.h, mytm.h): Millisecond ticker, periodic task callbacks, Timer class for elapsed-time measurement
- **Utilities** (csmacros.h, cskeys.h, csmisc.h): Macros (MIN/MAX/PIN), keyboard constants, tick counter, input polling

**Consumes from external systems:**
- **SDL:** Graphics surfaces, events, threading, mutex, `SDL_endian.h` macros (byte swapping), timer functions
- **Mac Carbon/Quickdraw:** Dialog APIs, graphics device manager, window/control management, event loop, resource system, timer manager
- **Resource system** (TextStrings.h): String and font resource retrieval and enumeration
- **Logging framework** (Logging.h): Error/warning output, logging domains
- **Standard C/C++ libraries:** `<streambuf>`, `<string>`, `<vector>`, `<stdio.h>`, `<cstdint>`, `<cstring>`
- **Generated config** (config.h): Build-time feature flags (`HAVE_OPENGL`, `HAVE_SNPRINTF`)

## Runtime Role

**Initialization:** `cseries.h` is the universal master include; platform-specific subsystems initialize via singleton patterns (e.g., `TimerInstaller` in timer.cpp installs the Time Manager task on Mac; SDL mutex created in mytm_sdl.cpp for thread-safe timer management).

**Frame/main loop:** String operations, dialog management, and timing services (mytm callbacks, timer.h) execute throughout frame processing; pixel format macros used in all rendering code; alert dialogs displayed on errors.

**Shutdown:** Resource cleanup in timer.cpp (memory unlocking on Mac), GWorld disposal in my32bqd.cpp, thread pool cleanup in mytm_sdl.cpp; fatal error shutdown via csalerts.h (ExitToShell on Mac, abort on SDL).

## Notable Implementation Details

- **Master header pattern:** `cseries.h` aggregates platform detection, type definitions, and includes to most CSeries headers; single point of configuration for the entire subsystem.
- **Platform-specific variants:** Conditional compilation gates implementations (_sdl.cpp, _macintosh.cpp, _carbon.cpp, _beos.cpp); cseries.h determines which is active.
- **Binary format durability:** `BStream` and `byte_swapping` use SDL's `SDL_SwapBE16/32/64` to ensure big-endian data (e.g., map files) serialize correctly across little-endian systems.
- **Dialog abstraction:** Two parallel implementationsΓÇöMac Carbon native (csdialogs_macintosh.cpp) and SDL widget emulation (csdialogs_sdl.cpp)ΓÇöboth expose the same QQ_* control API.
- **Resource-based localization:** Strings and fonts loaded from Mac resources via `TextStrings.h`; encoding convertors support Mac Roman (legacy), Unicode, and UTF-8.
- **Fixed-point arithmetic:** Macros like `INTEGER_TO_FIXED`, `FIXED_ONE`, `FIXED_FRACTIONAL_BITS` support 16.16 format used in coordinate calculations.
- **Performance telemetry:** `readout_data.h` exports global stat structures (timing, rendering counts, texture metrics) for runtime profiling and debug visualization.
- **Endianness transparency:** All multi-byte values serialized in big-endian; SDL macros handle swapping automatically on little-endian hosts.
