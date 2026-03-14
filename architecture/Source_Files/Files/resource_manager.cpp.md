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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `res_file_t` | struct | Encapsulates an open resource file with parsed type/ID maps |
| `id_map_t` | typedef | Maps resource ID (int) ΓåÆ file offset (uint32) for one resource type |
| `type_map_t` | typedef | Maps resource type (uint32) ΓåÆ id_map_t for all types in file |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `res_file_list` | `list<res_file_t*>` | static | Stack of open resource files |
| `cur_res_file_t` | `list<res_file_t*>::iterator` | static | Points to current active resource file |

## Key Functions / Methods

### is_applesingle
- **Signature:** `bool is_applesingle(SDL_RWops *f, bool rsrc_fork, int32 &offset, int32 &length)`
- **Purpose:** Detect AppleSingle format and locate resource fork.
- **Inputs:** SDL_RWops file handle; bool rsrc_fork (true for resource fork, false for data fork); reference outputs for offset/length.
- **Outputs/Return:** true if valid AppleSingle header found; offset and length populated.
- **Side effects:** Seeks file to position 0 and reads header.
- **Calls:** SDL_RWseek, SDL_ReadBE32, SDL_ReadBE16
- **Notes:** Recognizes AppleSingle versions with ID 0x00051600 and version 0x00020000.

### is_macbinary
- **Signature:** `bool is_macbinary(SDL_RWops *f, int32 &data_length, int32 &rsrc_length)`
- **Purpose:** Detect MacBinary II/III format and extract fork sizes.
- **Inputs:** SDL_RWops file handle; reference outputs for data/resource fork lengths.
- **Outputs/Return:** true if valid MacBinary header with correct CRC; data_length and rsrc_length populated.
- **Side effects:** Seeks file to position 0, reads 128-byte header, verifies CRC-16.
- **Calls:** SDL_RWseek, SDL_RWread
- **Notes:** Validates MacBinary CRC (0x1021 polynomial); supports versions up to 0x81.

### read_map
- **Signature:** `bool res_file_t::read_map(void)`
- **Purpose:** Parse resource map from the resource fork header and build type/ID lookup tables.
- **Inputs:** None (operates on member `f`).
- **Outputs/Return:** true if map successfully parsed; `types` map populated.
- **Side effects:** Seeks file extensively; logs anomalies/traces via logNote/logTrace/logAnomaly/logDump4.
- **Calls:** SDL_RWseek, SDL_RWtell, SDL_ReadBE32, SDL_ReadBE16; logging functions.
- **Notes:** Handles three file formats transparently: AppleSingle, MacBinary, raw fork. Validates resource header offsets/sizes and reference list integrity. Resource data offset is relative to fork start.

### open_res_file_from_rwops
- **Signature:** `SDL_RWops* open_res_file_from_rwops(SDL_RWops* f)`
- **Purpose:** Create res_file_t, parse resource map, and add to open file list; set as current.
- **Inputs:** SDL_RWops file handle (or NULL).
- **Outputs/Return:** Returns input file handle if successful; NULL if parse failed.
- **Side effects:** Allocates new res_file_t; closes file and returns NULL on map parse failure; adds to `res_file_list` and updates `cur_res_file_t` on success.
- **Calls:** read_map(); SDL_RWclose(); logNote1().
- **Notes:** Caller retains ownership of SDL_RWops handle on return (res_file_t does not call SDL_RWclose).

### open_res_file
- **Signature:** `SDL_RWops *open_res_file(FileSpecifier &file)`
- **Purpose:** Open a resource file, trying multiple file naming conventions and platform-specific approaches.
- **Inputs:** FileSpecifier reference with desired file path.
- **Outputs/Return:** SDL_RWops handle if successful; NULL otherwise.
- **Side effects:** Logs context; tries up to 5 path variants (.rsrc, .resources, raw, /..namedfork/rsrc, macOS resource fork).
- **Calls:** logContext1(), open_res_file_from_path(), open_res_fork_from_path() (macOS), sdl_rw_from_rfork() (BeOS), has_rfork_attribute() (BeOS).
- **Notes:** Platform-specific code paths for BeOS and macOS; falls back gracefully through variants.

### close_res_file
- **Signature:** `void close_res_file(SDL_RWops *file)`
- **Purpose:** Close a resource file and remove from active list.
- **Inputs:** SDL_RWops file handle.
- **Outputs/Return:** None.
- **Side effects:** Erases matching res_file_t from list; closes file; deletes res_file_t; updates `cur_res_file_t` to last file.
- **Calls:** find_res_file_t(), SDL_RWclose(), list::erase().
- **Notes:** No-op if file is NULL or not found in list.

### get_resource / get_1_resource / get_ind_resource / get_1_ind_resource
- **Signature:** `bool get_resource(uint32 type, int id, LoadedResource &rsrc)` (and variants)
- **Purpose:** Retrieve resource data by type+ID or type+index, searching current file or all open files.
- **Inputs:** Resource type (uint32), ID or index (int); output LoadedResource ref.
- **Outputs/Return:** true if resource found and loaded; false otherwise; rsrc populated on success.
- **Side effects:** Calls rsrc.Unload(); allocates memory with malloc(); seeks file and reads data; populates rsrc.p and rsrc.size (or calls SetData on macOS).
- **Calls:** Unload(), malloc(), SDL_RWseek(), SDL_ReadBE32(), SDL_RWread(); logging (commented out).
- **Notes:** `get_1_*` variants search only current file; multi-file variants iterate backward from current. Size is read as big-endian uint32 at resource data offset.

### count_resources / count_1_resources
- **Signature:** `size_t count_resources(uint32 type)` (and single-file variant)
- **Purpose:** Count resources of given type in current file or all open files.
- **Inputs:** Resource type (uint32).
- **Outputs/Return:** Count of resources.
- **Side effects:** None.
- **Calls:** find() and size() on type_map.
- **Notes:** Returns 0 if type not found. Multi-file variant iterates backward from current.

### has_resource / has_1_resource
- **Signature:** `bool has_resource(uint32 type, int id)` (and single-file variant)
- **Purpose:** Check if a resource exists without loading it.
- **Inputs:** Resource type (uint32), ID (int).
- **Outputs/Return:** true if exists; false otherwise.
- **Side effects:** None.
- **Calls:** type_map::find(), id_map::find().
- **Notes:** Multi-file variant iterates backward from current; stops and returns true on first match.

## Control Flow Notes
This module is initialization/file-loading code, not part of the frame loop. Resource files are opened once at startup or on-demand, kept in a static list, and queried during sprite/sound/map loading. The "current file" pattern mimics classic MacOS resource manager semantics where one fork is the active context.

## External Dependencies
- **SDL_RWops** (SDL.h, SDL_endian.h): Cross-platform file I/O abstraction.
- **FileSpecifier** (FileHandler.h): File path abstraction; used to generate candidate paths.
- **LoadedResource** (FileHandler.h): Output container for resource data.
- **Logging** (Logging.h): logNote, logTrace, logAnomaly, logDump4 for diagnostics.
- **Platform-specific symbols** (csfiles_beos.cpp): `has_rfork_attribute()`, `sdl_rw_from_rfork()` for BeOS; `open_fork_from_existing_path()` for macOS.
