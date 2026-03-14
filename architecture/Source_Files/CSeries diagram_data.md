# Source_Files/CSeries/BStream.cpp
## File Purpose
Implements binary serialization streams for reading and writing typed data with big-endian byte-order conversion. Provides concrete input/output stream classes wrapping `std::streambuf` to replace an older AStream implementation.

## Core Responsibilities
- Implement position tracking and stream size queries for input/output streams
- Read/write raw byte buffers with error checking and exception handling
- Deserialize integer and floating-point types from input streams with endianness conversion
- Serialize integer and floating-point types to output streams with endianness conversion
- Use SDL endianness macros to handle big-endian byte swapping on multi-byte values
- Throw `std::ios_base::failure` exceptions on bounds violations or incomplete reads/writes

## External Dependencies
- **Standard Library:** `<streambuf>`, `<ios_base>`, `<cstring>` (implicit for memcpy)
- **SDL:** `SDL/SDL_endian.h` for `SDL_SwapBE16()`, `SDL_SwapBE32()`, `SDL_SwapBE64()`
- **Project:** `cseries.h` (defines `uint8`, `int8`, `uint16`, `int16`, `uint32`, `int32`, `Uint64` type aliases)

# Source_Files/CSeries/BStream.h
## File Purpose
Provides binary stream serialization/deserialization classes that wrap C++ `streambuf` for reading and writing typed binary data. Intended as a replacement for `AStream`. Supports multiple integer/floating-point types with endianness handling (big-endian variant included).

## Core Responsibilities
- Abstract base for binary input/output operations via `streambuf`
- Define operator overloads for serializing/deserializing primitive types (int8, uint8, int16, uint16, int32, uint32, double)
- Support positional queries (`tellg`/`tellp`, `maxg`/`maxp`) on streams
- Provide concrete big-endian (BE) implementations for platform-specific byte order
- Expose raw byte read/write (`read()`, `write()`, `ignore()`)
- Manage streambuf attachment/detachment

## External Dependencies
- `cseries.h` ΓÇö defines `uint8`, `int8`, `int16`, `uint16`, `int32`, `uint32` (via `cstypes.h`)
- `<streambuf>` ΓÇö `std::streambuf`, `std::streampos`, `std::ios_base::failure`
- All integer typedefs defined elsewhere (not in this file)

# Source_Files/CSeries/byte_swapping.cpp
## File Purpose
Implements byte-swapping routines for endianness conversion on little-endian systems. Provides in-place swapping of 2-byte and 4-byte values, typically used during data serialization/deserialization (e.g., when reading/writing map files, network data, or other binary formats that may originate from big-endian systems).

## Core Responsibilities
- Swap byte order for 2-byte (16-bit) values in memory
- Swap byte order for 4-byte (32-bit) values in memory
- Support bulk field swapping via loop over multiple contiguous values
- Operate only on little-endian systems (conditional compilation)

## External Dependencies
- `cseries.h` (umbrella header for core types and SDL integration)
- `byte_swapping.h` (function declaration, enum definitions)
- `uint8` type (from `cstypes.h`, included transitively)
- Endianness detection via `ALEPHONE_LITTLE_ENDIAN` (derived from SDL's `SDL_BYTEORDER`)

# Source_Files/CSeries/byte_swapping.h
## File Purpose
Provides endianness conversion utilities for the Aleph One engine. Declares `byte_swap_memory()` to swap byte order in memory, with conditional compilation to optimize away the function on little-endian systems.

## Core Responsibilities
- Define type field markers for indicating byte counts during swaps
- Declare the primary byte-swapping function
- Conditionally disable swapping on systems that don't need it (little-endian)

## External Dependencies
- `<stddef.h>` (standard size/offset utilities)

# Source_Files/CSeries/csalerts.cpp
## File Purpose
Implements user-facing alert dialogs and error/assertion handling for the Aleph One game engine. Bridges platform-specific alert APIs (Carbon/Classic Mac) and logs messages to both UI and logging systems during runtime errors, warnings, and fatal shutdowns.

## Core Responsibilities
- Display modal alert dialogs with severity levels (info, fatal)
- Log errors, warnings, and assertions to both UI and logging system
- Handle assertion failures and debug warnings with file/line context
- Gracefully shut down the engine on fatal errors (cleanup + ExitToShell)
- Support dual platform implementations (Carbon vs. Classic Mac OS)
- Format error/assertion messages with csprintf and manage message buffers

## External Dependencies
- **Headers**: stdlib.h, string.h, Carbon/Carbon.h (conditional), csstrings.h, Logging.h, TextStrings.h
- **Macintosh APIs**: ExitToShell, Debugger, InitCursor, Alert, ParamText, NumToString (Classic); CopyCStringToPascal, StandardAlert (Carbon)
- **Utilities**: TS_GetCString (TextStrings), csprintf (csstrings), getpstr (csstrings), logError/logWarning/logFatal (Logging)
- **External symbols**: stop_recording() (defined elsewhere); ExitToShell marked NORETURN in non-MWERKS builds

# Source_Files/CSeries/csalerts.h
## File Purpose
Header file declaring alert, warning, and assertion infrastructure for the Aleph One game engine. Provides user-facing error dialogs, debug assertions, and panic/halt functions. Supports both debug and release builds with conditional macro implementations.

## Core Responsibilities
- Declare alert/dialog functions for displaying error messages to the user
- Define severity levels for categorizing alerts (info vs. fatal)
- Provide debug assertion macros that call internal assert/warn functions
- Declare halt/panic functions that terminate execution without returning
- Abstract platform-specific dialog behavior (Mac Carbon vs. SDL)
- Support both detailed (with message strings) and resource-based (with resid/item) alert variants

## External Dependencies
- `OSErr` (macOS error type; defined in Mac headers)
- `DialogItemIndex`, `AlertType` (Mac Carbon API types; conditional import)
- Compiler support for `__attribute__((noreturn))` (GCC/Clang; fallback for other compilers)
- Standard C library (const char*, int32)

# Source_Files/CSeries/csalerts_sdl.cpp
## File Purpose
SDL implementation of game alerts, assertions, and debugging support. Displays user-facing error/warning dialogs or console output depending on video availability, and provides assertion/halt handlers for the game engine.

## Core Responsibilities
- Display formatted alert dialogs (warnings/errors) with text wrapping to users
- Log alerts to the game logging system alongside user display
- Handle assertion failures with file/line context
- Provide pause/halt/debug breakpoint entry points
- Gracefully degrade to stderr output when SDL video is unavailable
- Manage program termination for fatal errors

## External Dependencies
- **SDL headers:** `<SDL.h>` (via cseries.h); provides video surface, dialogs, events
- **Logging system:** `Logging.h`; exports `logFatal()`, `logError2()`, `logWarning1()`, `logNote()`, `GetCurrentLogger()`, `logFatal1()`
- **Dialog/widget system:** `sdl_dialogs.h`, `sdl_widgets.h`; exports `dialog`, `vertical_placer`, `w_title`, `w_static_text`, `w_spacer`, `w_button`, `get_theme_font()`, `text_width()`, `dialog_ok`
- **Game engine:** `update_game_window()` (defined elsewhere), `stop_recording()` (defined elsewhere)
- **Utility:** `cseries.h` (master header); `<stdio.h>`, `<stdlib.h>` (abort, sprintf)
- **Defined elsewhere:** `getcstr()`, `csprintf()`, `font_info`, `MESSAGE_WIDGET`, `LABEL_WIDGET`, `TITLE_WIDGET`, `SPACER_WIDGET`, `BUTTON_WIDGET`, `dialog_ok` callback

# Source_Files/CSeries/cscluts.cpp
## File Purpose
Color lookup table (CLUT) management for Aleph One game engine on macOS. Converts between platform-agnostic `color_table` structures and macOS Carbon `CTabHandle` resource objects, while maintaining system color definitions (black, white, and UI colors).

## Core Responsibilities
- Convert `color_table` (engine format) to Macintosh `CTabHandle` (native format)
- Convert Macintosh `CTabHandle` (loaded from resources) to engine `color_table` format
- Initialize and expose system color constants (`rgb_black`, `rgb_white`, `system_colors`)
- Validate color counts (clamp to 0ΓÇô256 range) during conversion
- Allocate and populate Macintosh color specification arrays

## External Dependencies
- **Carbon.h:** `CTabHandle`, `ColorTable`, `ColorSpec`, `RGBColor`, `NewHandleClear()`, `GetCTSeed()`
- **FileHandler.h:** `LoadedResource` class, resource handle management
- **cscluts.h:** `color_table`, `rgb_color` type definitions; extern declarations for global colors

# Source_Files/CSeries/cscluts.h
## File Purpose
Defines color lookup table (CLUT) structures and initialization functions for the Aleph One game engine. Provides abstractions for managing color palettes on both Macintosh and cross-platform systems, supporting 256-color indexed graphics.

## Core Responsibilities
- Define color data structures (`rgb_color`, `color_table`) for palette representation
- Declare color table builder functions for both Mac-specific and generic implementations
- Define system color palette enum indices (highlight colors, grayscale tones)
- Declare global color constants (`rgb_black`, `rgb_white`, `system_colors[]`)
- Provide Mac Carbon bridge for native color table construction

## External Dependencies
- **Includes:** `cstypes.h` (integer types: `uint16`, `short`)
- **Forward declared:** `LoadedResource` class (resource management, defined elsewhere)
- **External symbols used:**
  - `CTabHandle` (Mac Carbon type, defined in Carbon headers)
  - `RGBColor` (Mac Carbon struct for color, defined in Carbon headers)
  - `NUM_SYSTEM_COLORS` (enum constant, `= 2` per enum definition)

# Source_Files/CSeries/cscluts_sdl.cpp
## File Purpose
Implements Mac CLUT (Color Look-Up Table) resource conversion for SDL. Provides global color constants and converts binary CLUT resource data into internal color_table structures with proper endianness handling.

## Core Responsibilities
- Define global color constants (black, white, system palette colors)
- Convert binary Mac CLUT resources to internal color_table format
- Handle big-endian byte order conversion for cross-platform compatibility
- Manage SDL_RWops streams for reading binary resource data
- Validate and clamp color count boundaries (0ΓÇô256)

## External Dependencies
- **Includes**: `cseries.h` (platform/debug macros), `FileHandler.h` (LoadedResource), `SDL_endian.h` (byte order)
- **SDL symbols used**: SDL_RWops (file stream), SDL_RWFromMem, SDL_RWseek, SDL_ReadBE16, SDL_RWclose
- **Symbols defined elsewhere**: RGBColor, color_table, rgb_color, NUM_SYSTEM_COLORS

# Source_Files/CSeries/csdialogs.h
## File Purpose
Header file providing cross-platform dialog management abstractions. It bridges Mac OS native dialog APIs and SDL emulations to enable shared dialog code between platforms. Defines type abstractions, macros, and function prototypes for creating, manipulating, and querying dialog controls (text fields, checkboxes, popups, etc.).

## Core Responsibilities
- Define platform-agnostic dialog pointer types (DialogPTR abstracts Mac WindowRef vs. SDL dialog class)
- Declare generic control manipulation API (QQ_* functions) available on all platforms
- Provide common text/number input/output utilities
- Declare Mac OS-specific dialog APIs (window framing, event filtering, control manipulation)
- Declare platform-specific implementations for specialized control types (boolean, selection, enabled/disabled state)
- Provide SDL emulation stubs for Mac OS Toolbox functions (HideDialogItem, ShowDialogItem)

## External Dependencies
- `<string>`, `<vector>`: Standard C++ containers for label lists
- **Mac OS frameworks** (conditional): Dialogs.h (via SDL_RFORK_HACK), Carbon window/control APIs
- **SDL**: Non-Mac implementations use custom dialog class
- **StringSet resources**: Referenced by ID for label population


# Source_Files/CSeries/csdialogs_macintosh.cpp
## File Purpose

Provides macOS dialog management and control manipulation for the Aleph One game engine. Wraps Mac OS Toolbox and Carbon APIs to abstract dialog creation, event handling, control manipulation, and modal dialog orchestration. Supports both classic Mac DITL-based dialogs and modern NIB-based interfaces.

## Core Responsibilities

- Dialog creation and initialization with standard button setup
- Control value extraction/insertion (numeric and text operations)
- Control state modification (hiliting, enabling/disabling, value setting)
- Dialog event filtering and window update handling
- User item frame drawing (region-based on classic, theme-based on Carbon)
- Modal dialog execution with event handler dispatch
- Popup menu population via callbacks
- Color picker integration with live preview
- Tab control pane visibility management
- Timer and event watcher lifecycle management
- Cross-platform control abstraction (QQ_* functions)

## External Dependencies

- **Includes (notable):**
  - `<Carbon/Carbon.h>` ΓÇô Mac OS Toolbox & Carbon APIs
  - `"csdialogs.h"` ΓÇô Dialog interface declarations
  - `"NibsUiHelpers.h"` ΓÇô NIB-specific helpers (AutoNibWindow, control text accessors)
  - `<string>`, `<vector>` ΓÇô STL containers for modern string/menu APIs

- **External symbols (defined elsewhere):**
  - `update_any_window()`, `activate_any_window()` ΓÇô Window event dispatchers
  - `build_stringvector_from_stringset()`, `pstring_to_string()`, `copy_string_to_pstring()` ΓÇô String utilities
  - `csprintf()`, `temporary`, `ptemporary` ΓÇô Debug sprintf
  - `vassert()` ΓÇô Debug assertion with message

# Source_Files/CSeries/csdialogs_sdl.cpp
## File Purpose
SDL-based dialog item manipulation utility providing compatibility wrappers between the Aleph One game engine's SDL dialog system and legacy Mac dialog APIs. Supplies simple functions to query and modify dialog widget properties (text, selection, enabled state) with minimal boilerplate. Created by Woody Zenfell (2001) to ease porting and maintenance across platforms.

## Core Responsibilities
- Hide/show dialog items by toggling widget enabled state
- Extract and insert numeric values into text entry widgets
- Modify widget selection and enabled state with optional parameter handling
- Query current selection/boolean state from dialog controls
- Convert between Pascal-strings (pstring) and C-strings for text field population
- Provide "QQ_" family of defensive functions that gracefully handle null widgets and multiple widget types
- Support dynamic label updates for selector widgets

## External Dependencies
- `cseries.h` ΓÇö umbrella header for Aleph One's C-series compatibility library (includes SDL, string/memory utilities, Mac type emulation)
- `sdl_dialogs.h` ΓÇö `dialog` class definition and theme constants
- `sdl_widgets.h` ΓÇö widget class hierarchy (`w_select`, `w_toggle`, `w_text_entry`, etc.)
- C standard library: `strlen()`, `strncpy()`, `strcpy()`, `free()`
- Aleph One utilities (defined elsewhere): `a1_p2cstr()`, `a1_c2pstr()`, `pstrdup()`, `build_stringvector_from_stringset()`

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

# Source_Files/CSeries/csfiles.cpp
## File Purpose
Provides macOS Classic/Carbon filesystem utilities for locating game data files and the running application executable. Handles file specification resolution with fallback path searching and bundle-aware application discovery.

## Core Responsibilities
- Resolve file specifications (FSSpec) from resource-based filenames and search paths
- Locate the running application's own file specification in the filesystem
- Support both Classic Mac and bundled application directory structures
- Perform path construction and file search iteration

## External Dependencies
- **Carbon.h** (macOS Classic/Carbon APIs: FSSpec, ProcessSerialNumber, ProcessInfoRec, GetProcessInformation, FSMakeFSSpec, kCurrentProcess)
- **csstrings.h** (`getpstr`, `countstr`)
- **csfiles.h** (own declarations)
- **<string.h>** (memcpy)

# Source_Files/CSeries/csfiles.h
## File Purpose
Header file declaring file system utility functions for the CSeries module. Provides an abstraction layer for retrieving file specifications in a platform-agnostic manner (likely targeting classic Mac OS). Part of the Aleph One game engine codebase.

## Core Responsibilities
- Declare file specification retrieval functions
- Define interface for accessing file system resources by list/item reference
- Support application file spec queries
- Abstract platform-specific file handling (FSSpec-based)

## External Dependencies
- `FSSpec` (Mac OS file specification type, defined elsewhere)
- `OSErr` (Mac OS error type, defined elsewhere)
- Code targets classic Mac OS era APIs; no modern platform headers visible

# Source_Files/CSeries/csfiles_beos.cpp
## File Purpose
BeOS-specific utility file for the Aleph One game engine. Provides directory discovery for application and preferences locations, and implements SDL-compatible I/O abstractions for reading/writing resource forks stored as BeOS file system attributesΓÇöenabling Marathon data files from Mac CD-ROMs to be used without conversion.

## Core Responsibilities
- Locate application and user preferences directories on BeOS via AppKit/StorageKit
- Detect resource fork attributes on files using BeOS filesystem APIs
- Wrap resource fork data in an SDL_RWops interface for transparent read/write access
- Manage file handle lifecycle and seek/read/write position tracking for resource fork streams

## External Dependencies
- **SDL:** `SDL_rwops.h`, `SDL_error.h` ΓÇô RWops abstraction and error handling.
- **BeOS AppKit:** `<AppKit.h>` ΓÇô `be_app`, `BEntry`, `BPath`, `app_info`.
- **BeOS StorageKit:** `<StorageKit.h>` ΓÇô `find_directory()`, path management.
- **POSIX:** `<unistd.h>`, `<fcntl.h>` ΓÇô file operations (`open`, `close`).
- **BeOS FS API:** `<fs_attr.h>` ΓÇô attribute stat/read/write (`fs_stat_attr`, `fs_read_attr`, `fs_write_attr`).
- **Standard:** `<string>` ΓÇô C++ string type.

# Source_Files/CSeries/csfonts.cpp
## File Purpose
Manages font resource loading and text specification configuration for Mac OS/Carbon platforms. Provides utilities to retrieve font specifications from resource files, get/set font properties on graphics ports, and manage TextSpec structures used throughout the engine for text rendering.

## Core Responsibilities
- Load TextSpec font specifications from Mac resource files ('finf' resources)
- Query current font settings from the active graphics port
- Apply font settings (font, style, size) to the graphics port
- Maintain a default/null TextSpec state
- Bridge between resource-based font configuration and runtime graphics operations

## External Dependencies
- **Mac OS APIs:** `<Carbon/Carbon.h>` (or fallback to `<Resources.h>`, `<Quickdraw.h>`)
  - Resource management: `GetResource()`, `LoadResource()`
  - Graphics port: `GetPort()`, `GetPortTextFont()`, `GetPortTextFace()`, `GetPortTextSize()`, `TextFont()`, `TextFace()`, `TextSize()`
- **Local:** `csfonts.h` (TextSpec struct definition), `cstypes.h` (int16, uint16 types)

# Source_Files/CSeries/csfonts.h
## File Purpose
Header file defining font and text styling types for the Aleph One game engine. Provides constants for text style flags (bold, italic, underline, shadow) and a `TextSpec` structure that encapsulates complete font configuration including file paths for different font variants.

## Core Responsibilities
- Define text styling bit flags (normal, bold, italic, underline, shadow)
- Define `TextSpec` struct for storing font metadata and variant paths
- Provide centralized font configuration schema used throughout the engine

## External Dependencies
- `cstypes.h`: Platform-specific integer type definitions (`int16`, `uint16`)
- `<string>`: Standard library for `std::string` (font file paths)


# Source_Files/CSeries/cskeys.h
## File Purpose
Defines keyboard key code constants for input handling throughout the game engine. Provides platform-independent abstraction of keyboard scancodes and special keys (navigation, function keys).

## Core Responsibilities
- Define standard ASCII-equivalent key codes (DELETE, HOME, END, arrow keys)
- Define extended/Macintosh key codes with `kx` prefix (ESCAPE, HELP, FWD_DELETE, PAGE_UP/DOWN)
- Define function key constants (F1ΓÇôF15)
- Provide consistent naming for keyboard input mapping across the codebase

## External Dependencies
- Only C preprocessor directives (`#ifndef`, `#define`, `#endif`)
- No external library dependencies


# Source_Files/CSeries/csmacros.h
## File Purpose
Utility header providing common mathematical, bitwise, and memory-manipulation macros and templates for the Aleph One game engine. Enables type-safe and bounds-checked operations on objects and arrays.

## Core Responsibilities
- Arithmetic macros (MIN, MAX, FLOOR, CEILING, PIN, ABS, SGN)
- Bit manipulation (FLAG operations for 16-bit and 32-bit flags)
- Object swapping (SWAP template)
- Bounds-checked array member access with null-pointer safety
- Type-safe wrappers for `memcpy` and `memset` eliminating manual `sizeof` calls
- Rectangle dimension calculations
- Power-of-two calculation

## External Dependencies
- `<string.h>` ΓÇö `memcpy`, `memset`

**Notes:**
- Macro-based bit operations are differentiated by width (16-bit vs. 32-bit); generic variants exist as well.
- Rectangle macros assume pointer-to-struct with `left`, `right`, `top`, `bottom` fields.
- All template functions guarded by `#ifdef __cplusplus` for C++ compatibility.

# Source_Files/CSeries/csmisc.cpp
## File Purpose
Platform abstraction layer providing cross-platform wrappers for system timing, input event polling, and debugger control on macOS (Carbon). Acts as a bridge between engine-neutral interfaces and platform-specific APIs.

## Core Responsibilities
- Provides system tick counter with platform-agnostic interface (`machine_tick_count`)
- Implements blocking wait for user input (click or keypress) with timeout
- Handles screen saver suppression (stub on macOS)
- Manages debugger initialization (stub implementation)
- Abstracts Carbon event loop APIs from engine code

## External Dependencies
- **Carbon/Carbon.h** ΓÇö macOS system framework; provides `TickCount()`, `GetNextEvent()`, `EventRecord`, and event masks (`mDownMask`, `keyDownMask`)
- **cstypes.h** ΓÇö defines `uint32` and portable integer types
- **csmisc.h** ΓÇö header declaring these functions
- Conditional compilation guards platform (`#if defined(mac)`, `EXPLICIT_CARBON_HEADER`)

# Source_Files/CSeries/csmisc.h
## File Purpose
Platform abstraction header providing timing, input, and utility functions for the game engine. Defines machine tick rates for different platforms (Macintosh, SDL) and declares core system-level functions including timing, input polling, and debug utilities.

## Core Responsibilities
- Define platform-specific timer constants (`MACHINE_TICKS_PER_SECOND`)
- Declare machine tick counter and timing functions
- Declare input event functions (click/keypress detection with timeout)
- Provide 68k assembly register access utilities (legacy Mac support)
- Screen saver management
- Debug initialization hook

## External Dependencies
- `uint32`, `bool`, `long` ΓÇö assume from standard C library (`<stdint.h>`, `<stdbool.h>`)
- Conditional compilation: `mac`, `SDL`, `env68k`, `DEBUG`
- No external library includes visible; assumes types defined elsewhere in CSeries

# Source_Files/CSeries/csmisc_sdl.cpp
## File Purpose
SDL-based implementation of miscellaneous utility functions for the Aleph One game engine. Provides cross-platform abstractions for system timing and input polling (mouse/keyboard detection).

## Core Responsibilities
- Get current system tick count (milliseconds since SDL init)
- Poll SDL event queue for mouse button or keyboard events
- Block execution with timeout waiting for user input

## External Dependencies
- SDL.h (timing, event polling)
- cseries.h (local game engine headers, defines uint32 and platform macros)

# Source_Files/CSeries/cspixels.h
## File Purpose
Defines pixel color type aliases and provides macros for converting between RGB color values and packed pixel formats (16-bit and 32-bit). Supports color component extraction from pixel values in both 16-bit and 32-bit formats.

## Core Responsibilities
- Define typed aliases for 8-bit, 16-bit, and 32-bit pixel representations
- Provide RGB-to-pixel conversion macros for 16-bit (5-5-5 RGB) format
- Provide RGB-to-pixel conversion macros for 32-bit (8-8-8 RGB) format
- Provide component extraction macros to read R, G, B values from packed pixels
- Define color-space constants (maximum values, component counts)

## External Dependencies
- `#include "cstypes.h"` ΓÇô Provides base type definitions (`uint8`, `uint16`, `uint32`)

---

**Notes:**
- Macro design assumes input values in range 0x0000ΓÇô0xFFFF; outputs depend on format (0x00ΓÇô0x1F for 16-bit, 0x00ΓÇô0xFF for 32-bit).
- 16-bit format uses bit-packing: bits 14ΓÇô10 (red), 9ΓÇô5 (green), 4ΓÇô0 (blue).
- 32-bit format uses byte alignment: bytes 2 (red), 1 (green), 0 (blue).

# Source_Files/CSeries/csstrings.cpp
## File Purpose

String and character encoding utility module for the Aleph One game engine. Provides resource-based string retrieval (Pascal and C strings), formatting functions, and bidirectional character encoding converters (Mac Roman Γåö Unicode Γåö UTF-8) to support legacy Mac string formats and international text.

## Core Responsibilities

- Retrieve strings from resource system (Pascal and C formats)
- Convert between Pascal strings (length-prefixed) and C strings (null-terminated)
- Perform sprintf-like formatting into Pascal and C string buffers
- Build std::vector/std::string from resource string sets
- Convert character encodings: Mac Roman Γåö Unicode, Unicode Γåö UTF-8
- Provide deprecated logging functions (dprintf, fdprintf) that delegate to Logger framework

## External Dependencies

- `<stdio.h>`, `<stdarg.h>`, `<string.h>` ΓÇô Standard C library
- `<map>`, `<string>`, `<vector>` ΓÇô STL containers
- `TextStrings.h` ΓÇô Resource string retrieval (TS_GetString, TS_GetCString, TS_CountStrings)
- `Logging.h` ΓÇô Logger framework (GetCurrentLogger, logMessageV, logDomain, logAnomalyLevel)
- `cstypes.h` ΓÇô Type definitions (uint16)
- Conditional: `<Carbon/Carbon.h>` (Mac Carbon target only)

# Source_Files/CSeries/csstrings.h
## File Purpose

Header file providing string manipulation and encoding utilities for the Aleph One game engine. Bridges legacy Pascal-style strings (Str255) and modern C/C++ strings with platform-specific encoding support (Mac Roman Γåö Unicode/UTF-8).

## Core Responsibilities

- Declare Pascal-style string operations (pstrcpy, pstrncpy, pstrdup)
- Declare C-style string wrappers around resource-based strings (getcstr, getpstr)
- Provide printf-like functions for debug output (dprintf, fdprintf, csprintf, psprintf)
- Support in-place string format conversion (C Γåö Pascal)
- Support character encoding conversions (Mac Roman Γåö Unicode/UTF-8)
- Bridge legacy string types with modern C++ containers (std::string, std::vector)
- Define global temporary string buffer for short-term allocations

## External Dependencies

- `cstypes.h` ΓÇô cross-platform integer type definitions (int8, uint16, uint32, etc.)
- `<string>`, `<vector>` ΓÇô C++ Standard Library
- Implicit: Mac Toolbox (Str255, SInt types) when compiled on Mac platform
- Implicit: GCC compiler features (format attribute) detected via `__GNUC__`


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

# Source_Files/CSeries/gdspec.cpp
## File Purpose

Manages graphics device (monitor/display) enumeration, configuration, and user selection for the Aleph One game engine. Provides functionality to find compatible displays, build device specifications, set color depth, and present a UI dialog for users to choose their preferred display and color mode.

## Core Responsibilities

- Enumerate and filter available graphics devices based on capability requirements
- Store and match graphics device specifications (resolution, color depth, slot ID)
- Modify graphics device color depth at runtime
- Provide two implementations of a device selection dialog (modern NIB-based and classic Mac OS versions)
- Handle device-to-UI coordinate mapping and scaling for visual display of monitor layout
- Manage platform differences between Carbon and classic Mac OS APIs

## External Dependencies

- **Mac OS Quickdraw & Graphics Device Manager:** `GDHandle`, `GetDeviceList()`, `GetNextDevice()`, `TestDeviceAttribute()`, `HasDepth()`, `SetDepth()`, `GetSlotFromGDevice()`.
- **Mac OS Dialog Manager:** `DialogPtr`, `GetDialogItem()`, `SetDialogItem()`, `ModalDialog()`, `myGetNewDialog()`, `DisposeDialog()`.
- **Mac OS Control Manager:** `ControlRef`, `ControlHandle`, `SetControlValue()`, `GetControlValue()`, `CreateUserPaneControl()`, `SetControlID()`, `Draw1Control()`, `EmbedControl()`.
- **Mac OS Carbon / Cocoa:** `WindowRef`, `WindowPort`, `IsWindowActive()`, `ShowWindow()`, `HideWindow()`, `ShowSheetWindow()`, `HideSheetWindow()`.
- **CoreFoundation:** `CFStringRef`, `CFSTR()`.
- **Custom CSeries headers:** `cseries.h` (basic types, macros), `shell.h` (game configuration).
- **Custom Mac UI helpers:** `NibsUiHelpers.h` (NIB loading, control helpers, modal dialog framework).
- **Standard library:** `<stdlib.h>`, `<vector>`.

# Source_Files/CSeries/gdspec.h
## File Purpose
Header file defining graphics device specification structures and functions for the Aleph One game engine. Provides abstractions for querying, matching, and configuring graphics device capabilities (resolution, bit depth, slot).

## Core Responsibilities
- Define the `GDSpec` structure to encapsulate graphics device parameters
- Declare functions for finding and matching compatible graphics devices
- Provide graphics device depth/bit-depth configuration and validation
- Support graphics device comparison and querying (slot, capabilities)
- Present a device selection dialog to the user

## External Dependencies
- `GDHandle` and `GDDevice` types: defined elsewhere (likely macOS Carbon API or engine-internal graphics system)
- Aleph One game engine copyright and GPL v2 license

# Source_Files/CSeries/my32bqd.cpp
## File Purpose
Provides a portability abstraction layer for Macintosh QuickDraw graphics operations, wrapping low-level GWorld and menu bar APIs. Abstracts differences between Classic Mac and Carbon/modern Mac implementations.

## Core Responsibilities
- Wrap QuickDraw GWorld (graphics world) creation, management, and disposal
- Provide pixel locking/unlocking and pixel map access abstractions
- Implement menu bar visibility control with platform-specific backends
- Stub out low-level color palette operations
- Provide initialization hook for graphics subsystem

## External Dependencies
- **Includes:** `Carbon/Carbon.h` (if `EXPLICIT_CARBON_HEADER` defined); `my32bqd.h` (header)
- **Conditional includes (Carbon only):** `csalerts.h` (for `dprintf`), `csstrings.h`
- **External symbols:** GWorld/GDevice/PixMap functions from Mac QuickDraw API; region and low-memory management functions (Classic only)
- **Defined elsewhere:** All wrapped Mac API functions (`GetGWorld`, `SetGWorld`, `NewGWorld`, `HideMenuBar`, etc.)

# Source_Files/CSeries/my32bqd.h
## File Purpose
Provides a C compatibility/abstraction layer for 32-bit QuickDraw graphics operations on classic and Carbon macOS. Encapsulates GWorld (offscreen graphics buffer) lifecycle and related graphics context management to abstract platform-specific graphics APIs.

## Core Responsibilities
- Initialize the 32-bit QuickDraw subsystem
- Manage graphics world (GWorld) creation, updating, and disposal
- Get/set the current graphics drawing context and device
- Lock/unlock pixel buffers for direct memory access
- Query graphics world pixel maps and base addresses
- Manage menu bar visibility
- Configure color palette entries

## External Dependencies
- `Carbon.h` or `QDOffscreen.h` ΓÇö macOS Carbon framework; provides `GWorldPtr`, `GDHandle`, `Rect`, `CTabHandle`, `PixMapHandle`, `ColorSpec`, `OSErr`.
- Conditional include suggests fallback compatibility paths for different macOS versions.

# Source_Files/CSeries/mytm.h
## File Purpose
Header file for a lightweight task scheduler that manages timed callbacks. Supports creation, removal, and reset of scheduled tasks with millisecond-level timing, thread-safe mutex access for multi-threaded synchronization, and periodic cleanup of abandoned threads.

## Core Responsibilities
- Declare task scheduling API (`myTMSetup`, `myXTMSetup`) for registering time-based callbacks
- Provide task lifecycle management (creation, removal, reset)
- Expose thread-safe mutex primitives for synchronizing access with other subsystems
- Supply periodic cleanup mechanism for reclaiming zombie thread resources
- Offer RAII-style mutex wrapper for exception-safe locking

## External Dependencies
- Standard C types: `int32`, `bool` (inferred from Aleph One codebase conventions)
- C++ `class` (RAII pattern for exception-safe locking)
- Task implementation details defined elsewhere

# Source_Files/CSeries/mytm_mac_carbon.cpp
## File Purpose
Implements a repeating timer system for macOS using Carbon Events API. Manages timer callbacks that execute at specified intervals, allowing callbacks to control timer lifecycle by returning true (continue) or false (stop).

## Core Responsibilities
- Create and install repeating timers via macOS Carbon Event Loop
- Execute user-provided callback functions at millisecond intervals
- Track timer state (primed/active vs. inactive)
- Remove timers and deallocate resources
- Support reinstalling inactive timers via reset operation

## External Dependencies
- **Carbon/Carbon.h:** Event Loop API (`GetMainEventLoop`, `InstallEventLoopTimer`, `RemoveEventLoopTimer`, `SetEventLoopTimerNextFireTime`, `EventLoopTimerRef`, `EventLoopTimerUPP`, `NewEventLoopTimerUPP`, `kEventDurationMillisecond`, `kDurationMillisecond`).
- **cstypes.h:** Type definitions (`int32`, `bool`).
- **csmisc.h:** Miscellaneous utilities (unused in this file).
- **mytm.h:** Function declarations exported to rest of codebase.

# Source_Files/CSeries/mytm_macintosh.cpp
## File Purpose
Implements a Macintosh-specific timer task abstraction for the Aleph One game engine, wrapping the native Timer Manager (TMTask) API. Provides periodic callback-based timers for both classic 68k Mac OS and Carbon frameworks, with platform-specific memory management handling.

## Core Responsibilities
- Wrap Macintosh Timer Manager tasks with custom callback function pointers
- Manage timer task lifecycle (setup, reset, removal)
- Handle platform-specific concerns (68k A5 register, Carbon UPP allocation)
- Execute periodic callback functions and manage re-priming
- Provide thread-safe mutex stubs for game engine integration

## External Dependencies
- **Macintosh APIs:** `<Timer.h>` (classic) or `<Carbon/Carbon.h>` (Carbon)ΓÇöprovides `TMTask`, `InsTime`, `InsXTime`, `PrimeTime`, `RmvTime`
- **CSeries headers:** `cstypes.h` (int32, uint32), `csmisc.h` (A5 register helpers on 68k)
- **Platform conditionals:** `env68k` (68k CPU), `TARGET_API_MAC_CARBON`, `EXPLICIT_CARBON_HEADER`
- **Defined elsewhere:** User callback functions, Timer Manager runtime

# Source_Files/CSeries/mytm_sdl.cpp
## File Purpose
Implements an SDL-based Time Manager emulation for the Aleph One engine, providing drift-corrected periodic timers that originally used MacOS Time Manager APIs. Used primarily by networking code for recurring callbacks.

## Core Responsibilities
- Initialize and manage a global SDL mutex serializing timer task execution
- Create worker threads that execute periodic callbacks with drift correction
- Manage task lifecycle (setup, removal, reset, cleanup)
- Track outstanding tasks and reclaim zombie threads during shutdown
- Support debug-mode profiling of timing accuracy
- Coordinate thread priorities for reliable execution

## External Dependencies
- **SDL:** `SDL_thread.h`, `SDL_timer.h` (threads, `SDL_GetTicks()`, `SDL_Delay()`), `SDL_error.h`
- **Game engine:** `thread_priority_sdl.h` (`BoostThreadPriority()`), `mytm.h` (public interface), `cseries.h` (types, macros), `Logging.h` (logging).
- **Standard library:** `<vector>`

# Source_Files/CSeries/NibsUiHelpers.h
## File Purpose
This header provides RAII utility wrappers and helper functions for managing macOS Carbon/Cocoa UI resources, particularly for Interface Builder (NIB) integration. It abstracts low-level Carbon APIs into high-level automatic resource-cleanup classes and convenience functions for dialog management, event handling, control access, and text I/O.

## Core Responsibilities
- NIB resource loading and automatic window creation with cleanup
- Control (UI widget) reference management and text field access (static/editable, Pascal/C strings)
- Event loop integration: timers, keyboard input, mouse clicks via RAII wrappers
- Modal and sheet dialog execution and lifecycle
- Custom drawing and hit-testing for user-defined controls
- Tab control switching with pane visibility management
- Color picking dialogs and control property getters/setters

## External Dependencies
- **Carbon/Core Foundation**: CFStringRef, IBNibRef, WindowRef, ControlRef, EventLoopTimer*, EventHandler*, RGBColor, OSType, OSStatus, ControlID, Str255, ConstStr255Param
- **STL**: `<memory>` (auto_ptr), `<string>`, `<vector>`
- **Boost**: `<boost/function.hpp>` (TimerCallback, ControlHitCallback, GotCharacterCallback typedefs)
- **Defined elsewhere**: `initialize_MLTE()`, control text I/O, color picking, menu building, dialog handling (all .cpp implementations)

# Source_Files/CSeries/readout_data.h
## File Purpose
Declares global performance profiling structures and statistics counters for engine subsystems. Provides extern declarations for runtime metrics covering timing, rendering operations, and OpenGL texture management.

## Core Responsibilities
- Define timing statistics structure for profiling engine phases (world, AI, rendering, etc.)
- Declare pre-rendering statistics counters (nodes, windows, objects)
- Declare rendering statistics counters (surfaces, primitives, pixels)
- Declare OpenGL texture statistics and binding metrics
- Export global stat instances for engine-wide telemetry collection

## External Dependencies
- **Includes:** `timer.h` ΓÇö provides `Timer` class for high-resolution timing
- **Extern definitions:** All four global stat variables are declared `extern` here but defined in another translation unit (likely engine core or main application file)

# Source_Files/CSeries/snprintf.cpp
## File Purpose
Provides platform-agnostic fallback implementations of `snprintf()` and `vsnprintf()` for systems lacking these standard C library functions. Wraps the system's `vsprintf()` with overflow detection and logging warnings.

## Core Responsibilities
- Implement `snprintf()` as a thin variadic wrapper (if `HAVE_SNPRINTF` undefined)
- Implement `vsnprintf()` using `vsprintf()` with buffer-overrun detection (if `HAVE_VSNPRINTF` undefined)
- Emit warnings to the logging system when formatted output exceeds buffer size
- Guard against recursive logging by tracking warning state

## External Dependencies
- **`<stdio.h>`** ΓÇô Standard I/O (implicitly via `vsprintf()`)
- **`<stdarg.h>`** ΓÇô Variadic argument handling (from snprintf.h)
- **`Logging.h`** ΓÇô `logWarning2()` function for overflow warnings
- **`snprintf.h`** ΓÇô Header with conditional declarations and preprocessor guards

# Source_Files/CSeries/snprintf.h
## File Purpose
A compatibility header providing portable declarations for `snprintf()` and `vsnprintf()` functions. It conditionally declares these functions if the platform lacks them, enabling safe formatted string writing with buffer-size limits across different systems.

## Core Responsibilities
- Conditionally declare `snprintf()` if not available on the platform
- Conditionally declare `vsnprintf()` (variadic version) if not available
- Provide a fallback wrapper for platforms missing standard C library implementations
- Shield the codebase from platform-specific availability differences

## External Dependencies
- `<stdarg.h>` ΓÇö for `va_list` type
- `<streambuf>` ΓÇö included as a workaround for MSVC 7.0 compatibility issue (unused in this file)
- `config.h` ΓÇö generated by Autoconf; defines `HAVE_SNPRINTF` and `HAVE_VSNPRINTF` macros (conditionally included)

# Source_Files/CSeries/timer.cpp
## File Purpose
Implements a Macintosh Classic timer system using the Time Manager toolbox. Maintains a global millisecond counter incremented by a platform timer interrupt and provides a Timer class for measuring elapsed time across application code.

## Core Responsibilities
- Manage global `globalTime` counter incremented by Mac Time Manager callbacks
- Install and initialize the Time Manager task on startup via singleton `TimerInstaller`
- Handle periodic timer interrupts and reschedule the timer task
- Provide `Timer::Clicks()` method to query elapsed time relative to a start point
- Clean up timer task and memory locks on shutdown

## External Dependencies
- **Mac Toolbox headers:** `<MacMemory.h>`, `<Timer.h>`, `<Gestalt.h>`
- **Mac Toolbox functions (defined elsewhere):**
  - `HoldMemory()`, `UnholdMemory()` ΓÇô lock/unlock memory pages
  - `Gestalt()` ΓÇô query OS capabilities
  - `InsXTime()`, `RmvTime()` ΓÇô install/remove Time Manager task
  - `PrimeTime()` ΓÇô schedule next timer interrupt
  - `NewTimerUPP()` ΓÇô create universal procedure pointer for callback

# Source_Files/CSeries/timer.h
## File Purpose
Provides a `Timer` class for measuring elapsed time intervals in a game engine. Tracks time using a global millisecond counter and allows accumulating multiple start/stop cycles. Includes a platform-specific warning about Mac crash handling.

## Core Responsibilities
- Declare a global monotonic time counter (`globalTime`) updated by the engine
- Provide a `Timer` class to measure elapsed time across one or more intervals
- Convert elapsed time from clicks (milliseconds) to seconds
- Accumulate total time from multiple Start/Stop cycles

## External Dependencies
- `#include <cstdint>` or equivalent (implicit; `uint32` type)
- `globalTime` declared elsewhere (engine core)
- `Clicks()` implemented elsewhere



