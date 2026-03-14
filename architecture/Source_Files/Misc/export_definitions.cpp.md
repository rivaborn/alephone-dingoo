# Source_Files/Misc/export_definitions.cpp

## File Purpose
Standalone command-line utility that exports game entity definitions (weapons, monsters, projectiles, effects, physics models) into a binary physics WAD file. Used during the build process to serialize hardcoded definition data into a loadable game asset format.

## Core Responsibilities
- Parse command-line arguments for destination file path
- Initialize file system and FSSpec structures
- Create an empty WAD container and populate it with definition records
- Iterate through all definition types and serialize their binary data
- Write WAD header, definition blocks, directory structure, and file checksum
- Validate file creation and report errors

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `wad_header` | struct | Metadata for WAD file format (version, checksum, layout offsets) |
| `directory_entry` | struct | Index entries describing where each WAD block is located |
| `wad_data` | struct | In-memory representation of WAD content (tags and data arrays) |
| `definition_data` | struct | Generic wrapper for a definition record (tag, data pointer, size, count) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `definitions` | definition_data[] (extern) | global | Array of definition records indexed by `NUMBER_OF_DEFINITIONS`; populated from included definition headers |

## Key Functions / Methods

### main
- **Signature:** `void main(int argc, char **argv)`
- **Purpose:** Entry point; validates command-line arguments and orchestrates file creation.
- **Inputs:** `argc`, `argv` (destination file path in argv[1])
- **Outputs/Return:** Exits with status code (0 on success, 1 on error)
- **Side effects:** Calls `initialize_debugger(true)`, `FSMakeFSSpec()`, file I/O
- **Calls:** `initialize_debugger()`, `get_my_fsspec()`, `FSMakeFSSpec()`, `create_physics_file()`
- **Notes:** Requires exactly one argument (destination path); converts path to Pascal string with `c2pstr()`; handles Macintosh FSErr codes

### create_physics_file
- **Signature:** `static bool create_physics_file(FileDesc *file)`
- **Purpose:** Core logic: create WAD file, serialize all definitions, write directory and checksum.
- **Inputs:** `file` ΓÇô pointer to FSSpec-style file descriptor
- **Outputs/Return:** `true` if successful, `false` otherwise
- **Side effects:** Creates/overwrites file; allocates WAD structures; I/O operations
- **Calls:** `create_wadfile()`, `open_wad_file_for_writing()`, `fill_default_wad_header()`, `write_wad_header()`, `create_empty_wad()`, `append_data_to_wad()`, `write_wad()`, `calculate_wad_length()`, `write_directorys()`, `calculate_and_store_wadfile_checksum()`, `close_wad_file()`, `read_wad_file_checksum()`
- **Notes:** Loops through `NUMBER_OF_DEFINITIONS` entries; each definition's binary data (size ├ù count bytes) is appended as a WAD tag; directory records offset and length of the single WAD block

## Control Flow Notes
This is a **one-shot build utility**, not part of the frame loop:
1. **Initialization:** Parse args, set up file system
2. **WAD Creation:** Create empty in-memory WAD, populate with all definitions
3. **Serialization:** Write header, definition block, directory entries
4. **Finalization:** Compute and store file checksum, close file
5. **Shutdown:** Exit process

Runs during development/build to pre-serialize definitions before game runtime.

## External Dependencies
- **Notable includes:**
  - `macintosh_cseries.h` ΓÇô platform abstraction
  - `wad.h` ΓÇô WAD file I/O (file creation, header/directory management, checksum)
  - Definition headers: `weapon_definitions.h`, `monster_definitions.h`, `projectile_definitions.h`, `effect_definitions.h`, `physics_models.h` (included with `#define EXPORT_STRUCTURE 1` to expose definitions)
  - Game system headers: `map.h`, `effects.h`, `projectiles.h`, `monsters.h`, `weapons.h`, `items.h`, `SoundManager.h`, `media.h`, `tags.h`

- **External symbols (defined elsewhere):**
  - `definitions` ΓÇô array of serializable definition records
  - `NUMBER_OF_DEFINITIONS` ΓÇô count of definition types
  - `CURRENT_WADFILE_VERSION`, `BUNGIE_PHYSICS_DATA_VERSION` ΓÇô version constants
  - `PHYSICS_FILE_TYPE` ΓÇô WAD typecode constant
  - `MAXIMUM_DIRECTORY_ENTRIES_PER_FILE` ΓÇô directory capacity constant
