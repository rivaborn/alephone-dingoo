# Source_Files/Files/tags.h

## File Purpose
Defines the tag system for the WAD file format used by Marathon/Aleph One. Provides typecode enumerations for different file types and four-character tag constants that identify chunks of game data (map geometry, save game state, physics, scripts, etc.). Supports platform-specific file type conversions on macOS.

## Core Responsibilities
- Define `Typecode` enum for abstracting file types across platforms
- Provide runtime getter/setter functions for typecodeΓåÆOS-type mappings
- Define four-character tag constants for WAD data chunk identification
- Support macOS resource fork loading and OS file type conversion
- Maintain separation between engine-level type abstraction and platform-specific file types

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Typecode` | enum | Abstracts file types (scenario, savegame, film, physics, shapes, sounds, images, preferences, music, etc.) across platforms |

## Global / File-Static State
None. The actual typecode values are loaded at runtime and stored elsewhere (`filetypes_macintosh.c`).

## Key Functions / Methods

### initialize_typecodes
- Signature: `void initialize_typecodes()`
- Purpose: Load typecodeΓåÆfile-type mappings from the macOS resource fork (or equivalent platform-specific storage)
- Inputs: None (reads from resource fork)
- Outputs/Return: None (populates internal typecode table)
- Side effects: Initializes runtime typecode state; must be called during engine startup
- Calls: Not inferable from this file
- Notes: Part of engine initialization sequence

### get_typecode
- Signature: `uint32 get_typecode(Typecode which)`
- Purpose: Look up the platform-specific file type for a given abstract typecode
- Inputs: `which` ΓÇö a `Typecode` enum value
- Outputs/Return: Platform-specific file type (e.g., OSType on Mac, uint32 generic)
- Side effects: None (pure lookup)
- Calls: Not inferable from this file
- Notes: Returns `'????'` for unrecognized typecodes (as of Aug 28, 2000 change)

### set_typecode
- Signature: `void set_typecode(Typecode which, uint32 _type)`
- Purpose: Store or override a typecode mapping at runtime
- Inputs: `which` ΓÇö typecode enum; `_type` ΓÇö platform-specific file type value
- Outputs/Return: None (modifies internal state)
- Side effects: Updates typecode table
- Calls: Not inferable from this file
- Notes: Added Jul 4, 2002 to support runtime configuration

### get_typecode_for_file_type (Mac only)
- Signature: `Typecode get_typecode_for_file_type(OSType inType)`
- Purpose: Reverse lookup: find typecode by OS file type
- Inputs: `inType` ΓÇö macOS OSType
- Outputs/Return: Matching `Typecode` enum value
- Side effects: None
- Calls: Not inferable from this file
- Notes: macOS-specific; guarded by `#ifdef mac`

### get_all_file_types_for_typecode (Mac only)
- Signature: `const std::vector<OSType> get_all_file_types_for_typecode(Typecode which)`
- Outputs/Return: Vector of OSType values (allows multiple OS types per typecode)
- Side effects: None
- Calls: Not inferable from this file
- Notes: macOS-specific; returns multiple types (e.g., classic + modern variants)

## Control Flow Notes
This is an initialization and data-definition module. `initialize_typecodes()` is called early in engine startup (likely before WAD loading). The accessors `get_typecode()` and `set_typecode()` are used throughout the game to abstract file I/O from platform-specific details. The tag constants (e.g., `POINT_TAG`, `PLAYER_STRUCTURE_TAG`) are embedded as four-character codes in WAD chunks and used during serialization/deserialization of map, game state, and physics data.

## External Dependencies
- `cstypes.h` ΓÇö provides `uint32`, `NONE` constant, `FOUR_CHARS_TO_INT()` macro
- `<vector>` ΓÇö C++ std::vector for Mac-specific multi-type returns
- macOS `OSType` (implicit when `#ifdef mac` is active)
