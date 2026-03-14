# Source_Files/Files/AStream.cpp
## File Purpose
Implements binary serialization/deserialization (packing/unpacking) with explicit endianness control. Provides four stream classes for converting primitive types to/from byte buffers while enforcing bounds checking and raising exceptions on overflow.

## Core Responsibilities
- Implement base I/O operators (`operator>>` for input, `operator<<` for output) for single-byte types (uint8, int8, bool)
- Provide Big-Endian (BE) and Little-Endian (LE) multi-byte deserialization/serialization with byte-order conversion
- Implement raw byte read/write and stream-skip operations
- Enforce buffer bounds validation and throw `AStream::failure` on overflow
- Define exception class with message handling for serialization errors
- Support method chaining via operator return-by-reference

## External Dependencies
- `AStream.h` ΓÇö class definitions, type aliases, exception spec
- `<string.h>` ΓÇö `memcpy()` for raw byte copying
- `<string>`, `<exception>` (via `AStream.h`) ΓÇö standard C++ exception base
- `cstypes.h` (via `AStream.h`) ΓÇö integer type definitions (uint8, int8, uint16, int16, uint32, int32)

# Source_Files/Files/AStream.h
## File Purpose
Provides templated binary stream classes for serialization and deserialization with explicit endian control. Replaces AlephOne's `Packing.h` with clearer API design, allowing runtime selection of Big Endian vs. Little Endian byte ordering through distinct stream types.

## Core Responsibilities
- Define templated base stream class with bounds checking and state management
- Provide input stream (`AIStream`) hierarchy for deserializing binary data
- Provide output stream (`AOStream`) hierarchy for serializing binary data
- Support runtime endian selection via `AIStreamBE`/`AIStreamLE` and `AOStreamBE`/`AOStreamLE` subclasses
- Manage stream state flags (good, fail, bad) and optional exception throwing on state changes
- Implement operator overloading for stream-style I/O syntax (`>>` and `<<`)
- Handle bulk read/write operations and byte skipping via `read()`, `write()`, `ignore()`

## External Dependencies
- **Standard Library:** `<string>` (for `std::string` in `failure` constructor), `<exception>` (base class `std::exception`)
- **Project headers:** `"cstypes.h"` (defines `uint8`, `int8`, `uint16`, `int16`, `uint32`, `int32`)
- **Defined elsewhere:** Actual implementations of `operator>>` / `operator<<` and `read()` / `write()` for integral types (in `.cpp` file, likely with endian swaps)

# Source_Files/Files/crc.cpp
## File Purpose
Implements CRC (Cyclic Redundancy Check) checksum generation for files and data buffers using lookup-table-based algorithms. Provides CRC32 and CCITT (16-bit) checksums for data integrity verification across file I/O and in-memory operations.

## Core Responsibilities
- Compute CRC32 checksums for files and memory buffers
- Compute CCITT (CRC16) checksums for memory buffers
- Build and manage a 256-entry lookup table for fast CRC32 computation
- Provide file I/O abstraction integration via `OpenedFile` handles
- Support incremental CRC computation on streamed file data

## External Dependencies
- **`FileHandler.h`** ΓÇô `FileSpecifier` (file path abstraction), `OpenedFile` (file I/O handle)
- **`cseries.h`** ΓÇô common type definitions (`uint32`, `uint16`, `int32`), macros, standard includes
- **`crc.h`** ΓÇô header declaring public API

# Source_Files/Files/crc.h
## File Purpose
Declares CRC (Cyclic Redundancy Check) calculation functions for data integrity verification. Provides interfaces for computing checksums on files (both by FileSpecifier and OpenedFile) and raw data buffers. Part of the Aleph One game engine's file I/O subsystem.

## Core Responsibilities
- Calculate 32-bit CRC for files identified by FileSpecifier reference
- Calculate 32-bit CRC for already-opened files (OpenedFile reference)
- Calculate 32-bit CRC for raw byte buffers
- Calculate 16-bit CRC-CCITT variant for raw byte buffers

## External Dependencies
- Forward declarations: `FileSpecifier`, `OpenedFile` (file abstraction classes defined elsewhere)
- Standard type definitions: `uint32`, `uint16`, `int32` (likely from platform headers)

# Source_Files/Files/extensions.h
## File Purpose
Header providing an interface for managing physics file loading and network synchronization in the Aleph One game engine. Declares functions for reading physics definitions from disk and serializing/deserializing physics data for network multiplayer.

## Core Responsibilities
- Set and manage the active physics file (user-selected or default)
- Load and parse physics definition structures from disk
- Serialize physics model data for network transmission
- Deserialize received network physics data

## External Dependencies
- **FileSpecifier** ΓÇô class representing file paths; defined elsewhere (object-oriented file handler per comment)
- **int32** ΓÇô platform abstraction for 32-bit integers (likely from platform headers)

# Source_Files/Files/FileHandler.cpp
## File Purpose
MacOS implementation of file and resource handling for the Aleph One game engine. Provides abstractions for low-level file I/O, resource fork management, directory navigation, and file dialogs, encapsulating MacOS FSSpec/resource fork APIs.

## Core Responsibilities
- File and resource file lifecycle management (open, close, read, write, position)
- MacOS resource fork operations (Push/Pop resource stack, load/check resources)
- Path parsing with Unix-style separators (/) translated to MacOS conventions
- Directory traversal and specification management
- File metadata queries (type, date, modification time, free space)
- File operations: creation, deletion, copying (with safe save via exchange), existence checks
- Navigation Services-based file dialogs (read, write, async write)
- MacBinary format detection and fork offset handling

## External Dependencies
- **MacOS APIs:** Carbon/Carbon.h (if EXPLICIT_CARBON_HEADER), FSSpec, FSRef, resource fork (Get1Resource, UseResFile, etc.), Navigation Services (NavGetFile, NavPutFile), Aliases, Folders.
- **Custom types:** `Typecode`, `OSType` (from tags.h); `DirectorySpecifier`, `OpenedFile`, `LoadedResource` (defined in FileHandler.h).
- **Utility functions:** `set_game_error`, `global_idle_proc`, `machine_has_nav_services`, `get_typecode`, `get_all_file_types_for_typecode` (defined elsewhere); `obj_clear`, `obj_copy`, `FOUR_CHARS_TO_INT`, `MIN`, `NONE` (from cseries.h).
- **SDL RWops:** Conditional support for SDL-based resource file I/O as alternative to native MacOS ResFile.

# Source_Files/Files/FileHandler.h
## File Purpose
Platform-agnostic abstraction layer for file, directory, and resource management in the Aleph One game engine. Provides unified C++ classes that wrap platform-specific I/O (MacOS resource forks vs. SDL/standard filesystem) to insulate the engine from platform details.

## Core Responsibilities
- Abstract low-level file operations (read, write, seek, close) via `OpenedFile`
- Manage in-memory resources (loading, unloading, access) via `LoadedResource`
- Handle resource fork files on MacOS via `OpenedResourceFile`
- Specify and manipulate file paths with type/creator awareness via `FileSpecifier`
- Provide directory specifications for MacOS volumes/parent IDs via `DirectorySpecifier`
- Support dialog-based file selection for opening/saving
- Enable file discovery, enumeration, and metadata queries (dates, types, free space)

## External Dependencies
- `tags.h` ΓÇô typecode definitions and helper macros (FOUR_CHARS_TO_INT, Typecode enum)
- `<stddef.h>` ΓÇô size_t
- `<time.h>` ΓÇô time_t (TimeType, presumably)
- `<vector>` ΓÇô STL vector for directory listings and resource lists
- `<string>` ΓÇô STL string (SDL only)
- `<SDL.h>` ΓÇô SDL_RWops file handle abstraction
- `<Carbon/Carbon.h>` ΓÇô MacOS Carbon framework (MacOS only)
- `<windows.h>` ΓÇô Windows API macros (Win32 only; undef GetFreeSpace, CreateDirectory to avoid conflicts)
- Platform-specific I/O not visible in this header (OSErr, FSSpec, refnum semantics defined elsewhere)

# Source_Files/Files/FileHandler_SDL.cpp
## File Purpose
SDL implementation of cross-platform file I/O for the Aleph One game engine. Provides abstractions for file operations, resource management, and file/directory selection dialogs with transparent handling of macOS resource forks and legacy file formats (AppleSingle, MacBinary).

## Core Responsibilities
- File I/O abstraction via SDL_RWops with position/length tracking and fork offset support
- Resource data lifecycle management (load, unload, detach)
- File path manipulation, type detection, directory operations across Windows/Unix/macOS
- Interactive file/directory browsing dialogs with sorting and navigation
- Automatic file type identification via magic bytes (Sounds, Maps/Scenarios, Physics, Shapes)
- File extension management and type code mapping
- Safe file operations (existence checks, overwrite confirmation, atomic exchange)

## External Dependencies
- **SDL:** `SDL_RWops`, `SDL_RWFromFile()`, `SDL_RWread/write/seek/tell/close()`, endian conversion functions
- **POSIX/Win32:** `stat()`, `mkdir()`, `access()`, `opendir/readdir/closedir()`, `rename()`, `remove()` (POSIX); `FindFirstFile/NextFile`, `GetLastError()` (Win32)
- **Engine:** `resource_manager.h` (resource fork operations), `shell.h` (directory specifiers), `game_errors.h` (error reporting), `SoundManager.h`, preferences system
- **UI:** `sdl_dialogs.h`, `sdl_widgets.h` (dialog/widget framework)
- **Other:** Boost string algorithms (`ends_with`), tags.h (typecode constants)

# Source_Files/Files/filetypes_macintosh.cpp
## File Purpose
Manages macOS file type codes (OSType) for the Aleph One game engine, mapping between the engine's abstract `Typecode` enum and macOS 4-byte file type codes. Loads custom typecodes from the resource fork and maintains backward compatibility with Marathon 2 file types through a multi-mapping system.

## Core Responsibilities
- Maintains a static array of OSType codes indexed by Typecode enum values
- Loads custom typecodes from macOS resource fork ('FTyp' resource 128) during initialization
- Builds and maintains a fast lookup map (std::map) from OSType ΓåÆ Typecode for file I/O operations
- Provides bidirectional accessors: Typecode ΓåÆ OSType and OSType ΓåÆ Typecode
- Supports one-to-many mappings (multiple OSTypes can map to one Typecode for M2 compatibility)
- Handles file types for scenarios, savefiles, physics, shapes, sounds, patches, images, preferences, and music

## External Dependencies
- **Carbon.h** (macOS APIs): Resource fork management (GetResource, HLock, HUnlock, ReleaseResource, GetHandleSize)
- **CSeries/csalerts.h:** Assert macro (used in get_typecode_for_file_type)
- **tags.h:** Typecode enum definition and function declarations
- **std::map, std::vector:** STL containers

# Source_Files/Files/find_files.cpp
## File Purpose
Implements the `FileFinder` class for recursively searching Mac OS directories for files matching a type criteria. Provides both buffer-based and callback-only search modes with support for directory change notifications.

## Core Responsibilities
- Clear FileFinder state (`Clear()`)
- Initiate recursive file search from a base directory (`Find()`)
- Recursively enumerate files and subdirectories in a directory tree (`Enumerate()`)
- Invoke file-matching callbacks and apply filtering based on type and flags
- Invoke directory entry/exit callbacks when recursing into subdirectories
- Support early termination via callback return values

## External Dependencies
- **Includes:** `"find_files.h"` (header), `"macintosh_cseries.h"` (Mac utilities), `<string.h>`, `<stdlib.h>`.
- **External symbols:**
  - `obj_clear()` ΓÇö memory utility macro/function.
  - `get_typecode()` ΓÇö converts type identifier to Mac OS typecode.
  - `PBGetCatInfo()` ΓÇö Mac OS API to retrieve directory/file catalog info.
  - `DirectorySpecifier`, `FileSpecifier`, `CInfoPBRec` ΓÇö defined elsewhere (FileHandler.h or Mac headers).
- **Conditional:** Code is only compiled on Mac (`#if defined(mac)`); SDL version exists in header as alternative class hierarchy.

# Source_Files/Files/find_files.h
## File Purpose
Provides cross-platform file-finding abstractions for both macOS (File Manager API) and SDL/generic platforms. Enables recursive directory traversal with type-based filtering and callback support for file discovery operations.

## Core Responsibilities
- Abstract file discovery across macOS FSSpec and SDL/cross-platform filesystem APIs
- Search directories for files matching a specified Typecode (file type)
- Support recursive traversal with subdirectory enumeration
- Buffer management for search results (count and destination array)
- Callback invocation for file filtering and directory change notifications
- Platform-specific error reporting

## External Dependencies
- **FileHandler.h**: Provides `FileSpecifier`, `DirectorySpecifier`, `Typecode` abstractions, and OpenedFile/OpenedResourceFile for file I/O
- **tags.h**: Defines typecode symbolic constants (`_typecode_unknown`, `_typecode_creator`, etc.)
- **macOS APIs** (conditional): `Files.h`, `Resources.h` ΓåÆ FSSpec, CInfoPBRec, OSErr, OSType, DirectoryID
- **SDL** (conditional): `<vector>` for result aggregation; canonicalized path handling
- **Standard C++**: `<vector>` for cross-platform result storage

# Source_Files/Files/find_files_sdl.cpp
## File Purpose
Implements SDL-based file discovery and enumeration for the Aleph One game engine. Provides recursive directory traversal with file type matching and abstract callback mechanisms for cross-platform asset location.

## Core Responsibilities
- Recursive directory traversal with optional type-code filtering
- Virtual callback pattern for flexible file discovery (abort-on-match or batch collection)
- Directory entry sorting (directories before files, then alphabetical)
- SDL platform abstraction for file system access

## External Dependencies
- **Includes:** `<vector>`, `<algorithm>` (for `std::sort`)
- **From FileHandler.h:** `DirectorySpecifier`, `FileSpecifier`, `Typecode`, `dir_entry`
- **From find_files.h:** `FileFinder` base class definition, `WILDCARD_TYPE` constant
- **Note:** Compiled only when `SDL_RFORK_HACK` is not defined (i.e., on SDL platforms, not native Mac with resource forks)

# Source_Files/Files/game_wad.cpp
## File Purpose
Handles loading and saving complete game maps and game state from/to WAD (Where's All the Data) files. Manages map file selection, level initialization, game state serialization, and network map distribution for the Aleph One engine.

## Core Responsibilities
- Select and open map files; manage current map file state
- Load level geometry and objects from WAD chunks
- Initialize new games with player placement and game parameters
- Build and serialize game state to save-game WAD files
- Export maps as playable WAD files
- Handle network map data transfer between players
- Restore previously saved games
- Query map metadata (entry points, checksums, level names)
- Manage physics model and Lua script loading

## External Dependencies
- **map.h** ΓÇö defines map structures (polygon, line, endpoint, etc.)
- **monsters.h, projectiles.h, effects.h** ΓÇö entity type definitions and packing/unpacking
- **player.h** ΓÇö player data and physics model info
- **network.h** ΓÇö networking function declarations
- **platforms.h** ΓÇö platform/door types
- **wad.h, FileHandler.h** ΓÇö WAD file I/O
- **Packing.h** ΓÇö serialization (pack/unpack) routines for game data
- **shell.h, preferences.h, ChaseCam.h, render.h** ΓÇö engine subsystems
- **XML_LevelScript.h, Music.h, SoundManager.h** ΓÇö script and audio management
- **computer_interface.h** ΓÇö terminal state management
- **game_errors.h, game_window.h, interface.h** ΓÇö UI and error reporting

# Source_Files/Files/game_wad.h
## File Purpose
Header declaring public interfaces for game WAD file operations in the Aleph One engine. Manages save/load functionality, level/map loading, and associated file specifications.

## Core Responsibilities
- Save and export game state to WAD files
- Load and process WAD files (maps, physics, scripting)
- Manage current map file context and file specifications
- Pause/resume game execution
- Query embedded level metadata (physics, Lua scripts)
- Validate map file integrity via checksums

## External Dependencies
- `FileSpecifier` class (OOP file handler; defined elsewhere)
- `wad_data` struct (WAD container; defined elsewhere)
- References to Chris Pruett's Pfhortran (scripting system)

# Source_Files/Files/import_definitions.cpp
## File Purpose
Manages loading and unpacking physics definition data from WAD files into the game engine. Handles initialization of physics-related structures (monsters, effects, projectiles, weapons, constants) for both single-player initialization and network multiplayer synchronization.

## Core Responsibilities
- Store and manage the active physics file specification
- Initialize all physics definition subsystems on game startup
- Load physics WAD data from disk or network sources
- Extract and unpack individual physics definition types from WAD containers
- Support network physics model transfer for multiplayer games
- Validate physics data versions and handle errors gracefully

## External Dependencies
- **WAD system:** `open_wad_file_for_reading`, `read_wad_header`, `read_indexed_wad_from_file`, `close_wad_file`, `extract_type_from_wad`, `free_wad`, `get_flat_data`, `inflate_flat_data` ΓÇö defined elsewhere (wad.h/wad.c)
- **Physics subsystems:** `init_monster_definitions`, `init_effect_definitions`, `init_projectile_definitions`, `init_physics_constants`, `init_weapon_definitions`, `unpack_monster_definition`, `unpack_effect_definition`, etc. ΓÇö defined elsewhere (monsters.h, effects.h, projectiles.h, weapons.h, physics_models.h)
- **File management:** `get_default_physics_spec` ΓÇö from interface.h / shell.h
- **Error handling:** `set_game_error` ΓÇö from game_errors.h
- **Tag constants:** `MONSTER_PHYSICS_TAG`, `EFFECTS_PHYSICS_TAG`, etc. ΓÇö from tags.h

# Source_Files/Files/mac_rwops.cpp
## File Purpose
Provides SDL_RWops callback implementations optimized for reading from classic Mac OS resource and data forks. Acts as a bridge between SDL's cross-platform Read/Write abstraction and the Mac OS File Manager API.

## Core Responsibilities
- Implement SDL_RWops seek/read/write/close callbacks for Mac file handles
- Convert SDL seek constants (RW_SEEK_SET/CUR/END) to Mac OS File Manager equivalents (fsFromStart/fsFromMark/fsFromLEOF)
- Provide factory function to open Mac file forks and wrap them in SDL_RWops
- Support both resource fork and data fork access on classic Mac OS files
- Manage lifecycle of Mac file handles (open/close)

## External Dependencies
- **SDL.h:** SDL_RWops, SDL_AllocRW, SDL_FreeRW, RW_SEEK_* constants
- **Files.h, Resources.h:** Mac OS File Manager APIs (FSSpec, FSMakeFSSpec, FSpOpenRF, FSpOpenDF, FSRead, FSClose, SetFPos, GetFPos)
- **cseries.h:** Project-wide definitions and includes

**Known issues (not inferable safety issues):**
- Operator precedence bug in `open_fork_from_existing_path`: the `!= noErr` check only applies to the FSpOpenDF branch, not FSpOpenRF
- Buffer overflow risk: strcpy used to build Pascal string without bounds checking

# Source_Files/Files/mac_rwops.h
## File Purpose
Header file declaring a utility function for opening macOS file forks as SDL RWops objects. Part of Aleph One's I/O subsystem, providing efficient read/write operations for classic Mac resource and data forks.

## Core Responsibilities
- Declare interface for opening resource/data forks from existing file paths
- Provide SDL-compatible I/O abstraction for Mac fork access
- Support platform-specific optimizations for Mac file fork handling

## External Dependencies
- **SDL**: `SDL_RWops` type (for read/write operations abstraction)
- **cseries.h**: Platform-specific types and SDL includes

# Source_Files/Files/Packing.cpp
## File Purpose
Implements byte-stream serialization/deserialization functions for converting 16-bit and 32-bit integers to/from packed byte sequences. Supports both big-endian and little-endian formats to handle the Marathon game data format and cross-platform compatibility.

## Core Responsibilities
- Convert uint16/int16 values to/from big-endian byte streams
- Convert uint32/int32 values to/from big-endian byte streams
- Convert uint16/int16 values to/from little-endian byte streams
- Convert uint32/int32 values to/from little-endian byte streams
- Advance stream pointers during read/write operations
- Provide signed/unsigned type overloads via function overloading

## External Dependencies
- `cseries.h` ΓÇö platform abstractions, SDL includes
- `cstypes.h` (included via cseries.h) ΓÇö typedef'd fixed-width integer types (uint8, uint16, uint32, int16, int32)
- `Packing.h` ΓÇö header declares public-facing macro names (`StreamToValue`, `ValueToStream`) that route to BE or LE versions

# Source_Files/Files/Packing.h
## File Purpose
Provides endianness-aware binary serialization/deserialization utilities for Marathon series game data. Converts between packed big-endian stream format (as stored in game files) and native host alignment, with optional little-endian support for toolchain compatibility.

## Core Responsibilities
- Serialize individual numerical values to byte streams (ValueToStream family)
- Deserialize individual numerical values from byte streams (StreamToValue family)
- Serialize/deserialize arrays of numerical values (ListToStream, StreamToList)
- Copy raw byte blocks to/from streams (BytesToStream, StreamToBytes)
- Support compile-time endianness selection (big-endian by default)
- Maintain stream pointer advancement during packing/unpacking operations

## External Dependencies
- `memcpy()` (C standard library)
- Standard integer types: `uint8`, `uint16`, `int16`, `uint32`, `int32` (defined elsewhere, likely platform headers)

# Source_Files/Files/preprocess_map_mac.cpp
## File Purpose
Mac-specific map and game file preprocessing routines. Handles locating default resource files (maps, physics, shapes, sounds) via recursive directory search, managing save game dialogs, and creating/embedding thumbnail previews in saved game files.

## Core Responsibilities
- Recursively search application directory for required game resource files by type code
- Present file dialogs for load/save game operations
- Generate and embed overhead map thumbnails in saved game files
- Add metadata resources (application name, etc.) to save files
- Initialize game state from selected save files
- Pause/resume game during file I/O operations

## External Dependencies
- **FileHandler.h** ΓÇô FileSpecifier and DirectorySpecifier classes; file dialog abstraction.
- **world.h** ΓÇô world_point2d, world_point3d, coordinate math.
- **overhead_map.h** ΓÇô overhead_map_data, _render_overhead_map().
- **game_wad.h** ΓÇô save_game_file(), get_current_saved_game_name().
- **Mac APIs** (via macintosh_cseries.h) ΓÇô FSSpec, GWorld, QuickDraw/Resource Manager (FSpOpenResFile, AddResource, etc.), Navigation Services.
- **shell.h** ΓÇô getcstr(), pause_game(), resume_game(), show/hide_cursor().

# Source_Files/Files/preprocess_map_sdl.cpp
## File Purpose
Save game and asset file management for SDL-based game engine. Provides utilities to locate default game assets (maps, physics, shapes, sounds, music, themes) and handle save/load game workflows via file dialogs.

## Core Responsibilities
- Locate default game asset files by searching data paths
- Distinguish between critical assets (map, shapes) and optional assets (physics, sounds, music)
- Display file dialogs for save game selection and confirmation
- Coordinate pause/resume and cursor visibility during save operations
- Provide extensibility hooks for platform-specific save file post-processing

## External Dependencies
- **cseries.h** ΓÇô Core engine infrastructure (macros, constants, string utilities)
- **FileHandler.h** ΓÇô FileSpecifier, DirectorySpecifier classes for cross-platform file I/O
- **world.h, map.h** ΓÇô Game world and map data structures
- **shell.h, interface.h** ΓÇô Game state and UI interface (pause_game, show_cursor, etc.)
- **game_wad.h** ΓÇô Game WAD serialization (save_game_file, get_current_saved_game_name)
- **game_errors.h** ΓÇô Error codes and alert macros (alert_user, fatalError, badExtraFileLocations)
- **vector** (STL) ΓÇô Used in data_search_path
- **data_search_path** (extern from shell_sdl.cpp) ΓÇô Multi-path search list for asset discovery

# Source_Files/Files/preprocess_map_shared.cpp
## File Purpose
Implements automatic game saving for Aleph One with intelligent filename generation. Provides functionality to save the current game state without user interaction, either overwriting the most recent save or creating a new uniquely-named file based on the current level name.

## Core Responsibilities
- Generate filesystem-safe filenames from level names (alphanumeric + spaces only)
- Create non-conflicting filename variants by appending numeric suffixes to prevent overwrites
- Execute automatic game saves with option to overwrite or create new files
- Report save results to the user via on-screen messages
- Handle platform-specific filename length constraints (Mac: 31 chars, others: 255 chars)

## External Dependencies
- **FileHandler.h**: `FileSpecifier`, `DirectorySpecifier` classes
- **game_wad.h**: `save_game_file()`, `get_current_saved_game_name()`
- **TextStrings.h**: `TS_GetCString()` (string resource lookup)
- **map.h**: `static_world->level_name` (current level name global)
- **shell.h**: `screen_printf()` (on-screen message display)
- **ctype.h**: `isalnum()` (character classification)
- Standard C/C++: `memset()`, `sprintf()`, `strlen()`, `strcmp()`, `strcpy()`

# Source_Files/Files/resource_manager.cpp
## File Purpose
Implements cross-platform MacOS resource file handling for the Aleph One engine. Supports reading resource data from AppleSingle, MacBinary II/III, and raw resource fork formats, managing multiple concurrent open resource files with a current-file stack model.

## Core Responsibilities
- Parse and validate MacOS resource file formats (AppleSingle, MacBinary, raw forks)
- Load resource type and ID maps from resource fork headers
- Maintain a list of open resource files with stack-based "current file" semantics
- Query resources by type/ID or type/index
- Allocate and return resource data as LoadedResource objects
- Support platform-specific resource fork access (BeOS attributes, macOS fork paths, regular files)

## External Dependencies
- **SDL_RWops** (SDL.h, SDL_endian.h): Cross-platform file I/O abstraction.
- **FileSpecifier** (FileHandler.h): File path abstraction; used to generate candidate paths.
- **LoadedResource** (FileHandler.h): Output container for resource data.
- **Logging** (Logging.h): logNote, logTrace, logAnomaly, logDump4 for diagnostics.
- **Platform-specific symbols** (csfiles_beos.cpp): `has_rfork_attribute()`, `sdl_rw_from_rfork()` for BeOS; `open_fork_from_existing_path()` for macOS.

# Source_Files/Files/resource_manager.h
## File Purpose
Header file providing cross-platform resource management API for the Aleph One game engine (Marathon port). Abstracts MacOS Classic resource fork handling using SDL for non-Mac platforms. Enables loading of game assets (graphics, sounds, data) from resource files.

## Core Responsibilities
- Initialize and manage resource file operations
- Open, close, and switch between resource files using SDL_RWops
- Count resources by type across single or chained resource files
- Retrieve resource IDs and enumerate resources by type or index
- Load individual resources by ID or index
- Query resource existence

## External Dependencies
- **stdio.h**: Standard C I/O (minimal usage in header)
- **vector (STL)**: Dynamic arrays for resource ID lists
- **SDL.h**: SDL library; provides SDL_RWops abstraction for file I/O
- **FileSpecifier** (defined elsewhere): File path abstraction
- **LoadedResource** (defined elsewhere): Resource data container

---

**Note:** The naming convention (`*_1_*` vs. `*_*`) suggests a Classic MacOS resource manager pattern: single-file functions search only the current file, while chain functions search across all open resource files in priority order.

# Source_Files/Files/tags.h
## File Purpose
Defines the tag system for the WAD file format used by Marathon/Aleph One. Provides typecode enumerations for different file types and four-character tag constants that identify chunks of game data (map geometry, save game state, physics, scripts, etc.). Supports platform-specific file type conversions on macOS.

## Core Responsibilities
- Define `Typecode` enum for abstracting file types across platforms
- Provide runtime getter/setter functions for typecodeΓåÆOS-type mappings
- Define four-character tag constants for WAD data chunk identification
- Support macOS resource fork loading and OS file type conversion
- Maintain separation between engine-level type abstraction and platform-specific file types

## External Dependencies
- `cstypes.h` ΓÇö provides `uint32`, `NONE` constant, `FOUR_CHARS_TO_INT()` macro
- `<vector>` ΓÇö C++ std::vector for Mac-specific multi-type returns
- macOS `OSType` (implicit when `#ifdef mac` is active)

# Source_Files/Files/wad.cpp
## File Purpose
Implements WAD (Where's All the Data) file I/O for the Marathon/Aleph One game engine. Provides serialization and deserialization of game resources (maps, textures, sounds, etc.) packaged into binary WAD archives, supporting multiple file format versions with backwards compatibility.

## Core Responsibilities
- Read WAD file headers and validate version/format compatibility
- Load indexed WADs from disk into in-memory `wad_data` structures (read-only or modifiable)
- Write WAD data with entry headers and directory information back to files
- Manage tag-based data queries (extract specific resource types from loaded WADs)
- Handle memory allocation strategy (level-transition vs. standard malloc)
- Support binary packing/unpacking with endianness conversion
- Calculate and verify WAD file checksums (CRC)
- Serialize/flatten WADs for network transfer
- Support multiple WAD file versions (Marathon 1 ΓåÆ Marathon Infinity)

## External Dependencies
- **FileHandler.h** ΓÇö `FileSpecifier`, `OpenedFile` file abstraction classes
- **Packing.h** ΓÇö `StreamToValue()`, `ValueToStream()`, endianness-aware binary (de)serialization
- **tags.h** ΓÇö Tag type definitions (e.g., `POLYGON_TAG`, `WadDataType`)
- **crc.h** ΓÇö `calculate_crc_for_opened_file()` for checksum computation
- **game_errors.h** ΓÇö `set_game_error()` for error reporting
- **interface.h** ΓÇö `alert_user()` for fatal memory errors; string resource `strERRORS`
- **cseries.h** ΓÇö Misc macros, platform abstractions, type definitions

# Source_Files/Files/wad.h
## File Purpose
Header file defining the binary WAD file format and API for reading, writing, and manipulating WAD containers (used in Marathon/Aleph One to store levels, sprites, textures, and other game data). Provides file I/O abstractions, in-memory data structures, and operations for loading, extracting, and modifying tagged game assets.

## Core Responsibilities
- Define binary WAD file format structures (headers, directories, entries) with version compatibility
- Declare functions for opening, closing, and navigating WAD files
- Provide API for reading and extracting tagged data from WADs
- Support creating, writing, and modifying WAD files in-place
- Validate file integrity via checksums and support WAD inheritance (parent-child relationships)
- Manage memory allocation for loaded WAD structures with between-levels mode control
- Abstract file I/O through FileSpecifier and OpenedFile classes

## External Dependencies
- `#include "tags.h"` ΓÇö Defines Typecode enum, tag constants (FOUR_CHARS_TO_INT macro), file type codes, typecode accessors
- Forward declarations: `class FileSpecifier;` and `class OpenedFile;` ΓÇö Cross-platform file abstraction (implementation elsewhere)
- Implicit: Standard C types (int16, int32, uint32, byte, char) from cstypes.h (included via tags.h)

# Source_Files/Files/wad_macintosh.cpp
## File Purpose

Provides platform-specific file search functionality for locating WAD (game resource) files in the Aleph One Marathon engine. Enables searching by checksum or modification date with recursive directory traversal and callback-based result filtering.

## Core Responsibilities

- Search for WAD files matching a specific checksum value
- Locate files matching a specific modification date timestamp
- Provide callback-based file enumeration integration with the `FileFinder` abstraction
- Support recursive directory searching with typecode-based file type filtering
- Confine searches to the application's root directory and subdirectories
- Handle error reporting for failed enumeration operations

## External Dependencies

- **Headers (portable abstractions):**
  - `wad.h` ΓÇô WAD file format, checksum validation (`wad_file_has_checksum()`, `wad_file_has_parent_checksum()`)
  - `tags.h` ΓÇô Typecode enum (`_typecode_*` constants)
  - `find_files.h` ΓÇô `FileFinder` class for file enumeration
  - `FileHandler.h` ΓÇô `FileSpecifier`, `DirectorySpecifier` classes, `Files_GetRootDirectory()`
  - `macintosh_cseries.h` ΓÇô Macintosh platform utilities
  - `game_errors.h` ΓÇô Error constants (included but unused)
  - `<string.h>` ΓÇô Standard C (included but unused in visible code)

- **Symbols defined elsewhere:**
  - `wad_file_has_checksum(FileSpecifier&, uint32)` ΓÇô checks WAD file checksum
  - `wad_file_has_parent_checksum(FileSpecifier&, uint32)` ΓÇô checks parent checksum (disabled code path)
  - `dprintf()` ΓÇô debug/error printing
  - `TimeType` type ΓÇô modification date/time representation

# Source_Files/Files/wad_prefs.cpp
## File Purpose
Manages persistent preferences storage using the WAD (Where's All the Data?) file format. Provides functions to initialize, load, retrieve, and persist game preferences to disk with automatic recovery from corruption or missing files.

## Core Responsibilities
- Opens and initializes the preferences WAD file at startup, platform-specific file paths
- Loads preferences from disk into a WAD structure in memory
- Retrieves typed preference data items with optional initialization and validation callbacks
- Writes modified preferences back to disk with atomic file replacement
- Handles file errors gracefully by creating empty preference containers
- Manages the global preferences state and validates preference data integrity

## External Dependencies
- **Notable includes:** `cseries.h` (platform abstractions), `wad.h` (WAD format and I/O), `game_errors.h` (error codes), `FileHandler.h` (FileSpecifier/OpenedFile classes, inferred)
- **External symbols:** `extract_type_from_wad`, `append_data_to_wad`, `create_empty_wad`, `free_wad`, `read_indexed_wad_from_file`, `calculate_wad_length`, `fill_default_wad_header`, `write_wad_header`, `write_directorys`, `write_wad`, `open_wad_file_for_reading`, `open_wad_file_for_writing`, `close_wad_file`, `read_wad_header`, `set_indexed_directory_offset_and_length` (from wad module); `set_game_error`, `get_game_error`, `error_pending` (from game_errors module); `FileSpecifier`, `OpenedFile` platform file handlers

# Source_Files/Files/wad_prefs.h
## File Purpose
Defines the public interface for preferences file management in the Aleph One game engine. Provides functions to open, read, and write preference data stored in WAD (Where's All the Data) format, with extensible callback-based initialization and validation. Mac-only UI code for preferences dialogs is also declared.

## Core Responsibilities
- Open and manage preferences files using FileHandler abstraction
- Read preference data blocks by type tag with size validation
- Write modified preferences back to disk
- Support custom initialization callbacks for allocation
- Support custom validation callbacks for data integrity
- (Mac) Declare dialog-based preferences UI with item-hit and teardown handlers
- Prevent preference data duplication via callback mechanism

## External Dependencies
- **FileHandler.h:** `FileSpecifier`, `OpenedResourceFile`, `OpenedFile` abstractions
- **tags.h:** `Typecode` enum, `WadDataType` enum (referenced but not included in this file)
- **Implicit WAD code:** Functions assume a loaded `wad_data` structure (type defined elsewhere)
- **(Mac-only):** Carbon/Toolbox dialog APIs (`DialogPtr`, `Typecode`, resource DITL/STR# management)

# Source_Files/Files/wad_prefs_macintosh.cpp
## File Purpose
Implements a Macintosh Carbon preferences dialog that allows users to navigate between different preference sections (e.g., graphics, sound, controls) via a popup menu and configure settings within each section. Remembers the currently selected section across invocations.

## Core Responsibilities
- Create and manage a modal preferences dialog with dynamically loadable sections
- Populate a popup menu with preference section names from resource strings
- Switch between preference sections, handling setup/teardown of UI elements
- Process user interactions: section selection, OK/Cancel, keyboard navigation
- Handle keyboard shortcuts (Page Up/Down, Home/End) for section navigation
- Persist the current section selection across dialog invocations
- Validate teardown operations before dismissing the dialog

## External Dependencies
- **Notable includes:** `cseries.h` (core macOS/Carbon compatibility), `map.h`, `wad.h` (WAD file structures), `game_errors.h`, `shell.h` (dialog resource constants), `wad_prefs.h` (callback struct definition)
- **External symbols:** `prefInfo` (extern), `myGetNewDialog`, `getpstr`, `ModalDialog`, `AppendMenu`, `SetMenuItemText`, `GetControlPopupMenuHandle`, `GetControlValue`, `SetControlValue`, `SetControlMaximum`, `CountDITL`, `ShortenDITL`, `AppendDITL`, `GetResource`, `ReleaseResource`, `ShowWindow`, `NewModalFilterUPP`, `DisposeModalFilterUPP`, `DisposeDialog`, `GetDialogItem`, `general_filter_proc`, `PIN` macro, dialog constants (`dlogPREFERENCES_DIALOG`, `iPREF_SECTION_POPUP`, `iOK`, `iCANCEL`, `NUMBER_VALID_PREF_ITEMS`)

# Source_Files/Files/wad_sdl.cpp
## File Purpose
SDL-based implementation for locating game map (WAD) files within configurable search paths. Provides two search strategies: by embedded checksum value or by modification date. Acts as a utility layer for the resource discovery pipeline.

## Core Responsibilities
- Locate WAD files matching a 32-bit checksum embedded at file offset 0x44
- Locate files matching a specific modification date
- Iterate through a global data search path (`data_search_path`) to find files
- Use template-style file finder pattern (FileFinder subclasses) to abstract search logic

## External Dependencies
- **SDL**: `SDL_endian.h` (SDL_ReadBE32 for endian-safe big-endian integer reads)
- **FileHandler.h**: FileSpecifier, OpenedFile, Typecode, TimeType, DirectorySpecifier abstractions
- **find_files.h**: FileFinder base class defining the callback-based enumeration pattern
- **cseries.h**: General utility types and platform macros
- **shell_sdl.cpp** (external): defines `data_search_path` vector


