# Subsystem Overview

## Purpose

The Files subsystem manages all file I/O operations in the Aleph One engine, providing cross-platform abstractions for reading and writing game data, assets, and persistent state. It implements binary serialization with endianness control, resource file handling (including macOS resource forks), WAD file format containers, and asset discovery pipelines for maps, physics definitions, sounds, and save games.

## Key Files

| File | Role |
|------|------|
| AStream.h / AStream.cpp | Templated binary stream classes with explicit big-endian/little-endian serialization, bounds checking, and exception handling |
| Packing.h / Packing.cpp | Byte-stream serialization utilities for converting 16/32-bit integers to/from packed sequences |
| FileHandler.h / FileHandler.cpp | macOS platform-specific file and resource abstraction (FSSpec, resource forks, dialogs) |
| FileHandler_SDL.cpp | Cross-platform file I/O using SDL_RWops, POSIX/Win32 APIs, and dialogue frameworks |
| resource_manager.h / resource_manager.cpp | MacOS resource file parser supporting AppleSingle, MacBinary II/III, and raw fork formats |
| wad.h / wad.cpp | WAD file format (Where's All the Data) for serializing game assets with versioning and checksums |
| game_wad.h / game_wad.cpp | Game state and map loading/saving, level initialization, network map distribution |
| tags.h | Typecode enum and four-character tag constants for WAD data chunks; platform-specific file type conversion |
| crc.h / crc.cpp | CRC32 and CCITT (CRC16) checksum computation for data integrity validation |
| find_files.h / find_files.cpp / find_files_sdl.cpp | Recursive file discovery with type-based filtering and callback-based enumeration |
| import_definitions.cpp | Physics definition loading and unpacking from WAD files |
| preprocess_map_mac.cpp / preprocess_map_sdl.cpp / preprocess_map_shared.cpp | Asset search, save/load dialogs, automatic game saves with thumbnail generation |
| filetypes_macintosh.cpp | Bidirectional mapping between engine Typecode enum and macOS OSType file type codes |
| extensions.h | Physics file management (set active file, load, serialize for network) |
| wad_prefs.h / wad_prefs.cpp | Preferences persistence using WAD format with callback-based initialization/validation |
| wad_prefs_macintosh.cpp | macOS Carbon preferences dialog UI |
| wad_sdl.cpp / wad_macintosh.cpp | Platform-specific WAD file search by checksum or modification date |
| mac_rwops.h / mac_rwops.cpp | SDL_RWops wrapper for macOS resource and data fork access |

## Core Responsibilities

- Abstract platform-specific file operations (macOS FSSpec/Carbon, Windows API, POSIX) via FileSpecifier, DirectorySpecifier, and OpenedFile classes to enable cross-platform code
- Implement binary serialization with explicit endianness control for converting game structures (maps, entities, physics) to/from byte streams
- Parse and manage MacOS resource files (native forks, AppleSingle, MacBinary II/III formats) with multi-file stacking semantics
- Define and implement WAD file format for storing and retrieving tagged game assets (geometry, textures, scripts, save state) with version-aware backwards compatibility
- Discover, locate, and load game assets (maps, physics definitions, shapes, sounds, music) from configurable search paths with type-based filtering
- Persist game state (current level, player position, inventory) to WAD files with automatic filename generation and save slot management
- Validate data integrity using CRC32 checksums for file I/O and network map distribution
- Map abstract file type codes (Typecode enum) to platform-specific file type identifiers (macOS OSType) and file extensions

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- **FileHandler API:** FileSpecifier, DirectorySpecifier, OpenedFile, OpenedResourceFile classes for uniform file operations across platforms
- **WAD API:** `open_wad_file_for_reading/writing()`, `read_wad_header()`, `extract_type_from_wad()`, `append_data_to_wad()`, `free_wad()` for asset container management
- **Game persistence:** `save_game_file()`, `load_game_state()`, automatic game saves via `do_auto_save()`
- **Asset loading:** `import_definitions_from_file()` for physics, `FileFinder` for recursive asset discovery
- **Binary serialization:** AStream, Packing utilities for all inter-subsystem data packing/unpacking
- **Type mapping:** `get_typecode_for_file_type()`, `get_file_type_for_typecode()` for file format abstraction
- **Preferences:** `open_preferences_file()`, `get_preference()`, `write_preferences()` for persistent settings

**Consumes from other subsystems:**
- **Game world (world.h, map.h):** Serializes geometry (polygons, lines, endpoints) and level metadata
- **Physics (monsters.h, effects.h, projectiles.h, weapons.h):** Unpacks physics definitions via `unpack_*_definition()` callbacks
- **Player/entities (player.h):** Saves/restores player state and initial placement
- **Networking (network.h):** Distributes maps to remote players via serialized WAD data
- **Shell/UI (shell.h, interface.h):** File dialogs, menu callbacks, error alerts
- **Rendering (render.h, overhead_map.h):** Generates and embeds map thumbnails in save files
- **Audio (SoundManager.h, Music.h):** Locates sound and music asset paths during initialization
- **Scripting (XML_LevelScript.h):** Loads Lua/script data from WAD containers

## Runtime Role

- **Init:** `preprocess_map_*()` searches for default asset files (maps, physics, shapes, sounds); `import_definitions_from_file()` unpacks physics definitions into game structures
- **Frame:** Game state saved explicitly via `save_game_file()` or automatically during level transitions via `do_auto_save()`
- **Shutdown:** Preferences and game state flushed to disk via `write_preferences_wad()` and `close_wad_file()`

## Notable Implementation Details

- Binary format enforces **big-endian byte order exclusively** (Marathon 1/2/Infinity standard; defined in Packing.h with optional LE variants)
- Resource file handling bridges three formats: native macOS resource forks (via FSSpec), AppleSingle (single-fork-portable format), and MacBinary II/III (legacy Mac transfer format)
- WAD format is versioned to support **Marathon 1 ΓåÆ Marathon Infinity backwards compatibility** with graceful format detection
- File type abstraction (Typecode enum) enables **asset discovery independent of platform file extensions**ΓÇöcritical for cross-platform data portability
- **Dingoo port specifics:** SDL-based implementation (FileHandler_SDL.cpp, find_files_sdl.cpp, preprocess_map_sdl.cpp) replaces Carbon/FSSpec calls; game state isolated per episode via `ALEPHONE_USERDATA` environment variable for constrained 32 MB RAM storage
- Checksum-based file validation (`crc.cpp`, `wad_file_has_checksum()`) enables robust network map distribution and scenario integrity checking
- Preferences stored in WAD format with callback-based initialization, allowing subsystems to define default values without hardcoding storage logic
- Platform-specific type mappings (filetypes_macintosh.cpp) maintain one-to-many OSType ΓåÆ Typecode relationships for Marathon 2 compatibility
