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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FileSpecifier | class (forward decl.) | Abstraction for file paths; used to specify which resource file to open |
| LoadedResource | class (forward decl.) | Container for loaded resource data returned by get_*_resource functions |
| SDL_RWops | struct (external) | SDL abstraction for file I/O; holds open resource file handle |
| vector<int> | template (STL) | Dynamic list of resource IDs retrieved from a resource file |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| (implicit) current resource file | SDL_RWops* | static/singleton | Maintained by `use_res_file()` and queried by `cur_res_file()` |
| (implicit) resource search chain | chain of SDL_RWops* | static | Resources searched across multiple open files in priority order |

## Key Functions / Methods

### initialize_resources
- Signature: `void initialize_resources(void)`
- Purpose: Initialize resource subsystem at startup
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up internal state for resource file handling
- Calls: (implementation not visible in header)
- Notes: Must be called before any other resource functions

### open_res_file
- Signature: `SDL_RWops *open_res_file(FileSpecifier &file)`
- Purpose: Open a resource file and add it to the resource search chain
- Inputs: `FileSpecifier` reference specifying the file path
- Outputs/Return: SDL_RWops handle for the opened file (or null on failure)
- Side effects: Opens file, allocates SDL_RWops structure, adds to search chain
- Calls: (implementation not visible)
- Notes: File remains open until `close_res_file()` is called

### use_res_file
- Signature: `void use_res_file(SDL_RWops *file)`
- Purpose: Set the current (primary) resource file for resource lookups
- Inputs: SDL_RWops handle from `open_res_file()` or `open_res_file_from_rwops()`
- Outputs/Return: None
- Side effects: Changes the current resource file context
- Calls: (implementation not visible)
- Notes: Affects single-file functions (`get_1_resource`, `count_1_resources`, etc.)

### count_1_resources / count_resources
- Signature: `size_t count_1_resources(uint32 type)` / `size_t count_resources(uint32 type)`
- Purpose: Count resources of a specific type
- Inputs: 4-byte resource type code (e.g., 'PICT', 'snd ')
- Outputs/Return: Count of resources matching type
- Side effects: None
- Calls: (implementation not visible)
- Notes: `count_1_resources` searches only current file; `count_resources` searches entire chain

### get_1_resource / get_resource
- Signature: `bool get_1_resource(uint32 type, int id, LoadedResource &rsrc)` / `bool get_resource(uint32 type, int id, LoadedResource &rsrc)`
- Purpose: Retrieve a specific resource by type and ID
- Inputs: Resource type, resource ID, reference to output structure
- Outputs/Return: `true` if found and loaded, `false` otherwise; resource data in `rsrc`
- Side effects: Loads resource data into memory
- Calls: (implementation not visible)
- Notes: `get_1_resource` searches only current file; `get_resource` searches entire chain

### get_1_ind_resource / get_ind_resource
- Signature: `bool get_1_ind_resource(uint32 type, int index, LoadedResource &rsrc)` / `bool get_ind_resource(uint32 type, int index, LoadedResource &rsrc)`
- Purpose: Retrieve a resource by enumeration index (0-based)
- Inputs: Resource type, 0-based index, reference to output structure
- Outputs/Return: `true` if found and loaded, `false` otherwise; resource data in `rsrc`
- Side effects: Loads resource data into memory
- Calls: (implementation not visible)
- Notes: Used to iterate over resources of a given type

### has_1_resource / has_resource
- Signature: `bool has_1_resource(uint32 type, int id)` / `bool has_resource(uint32 type, int id)`
- Purpose: Query whether a resource exists without loading it
- Inputs: Resource type, resource ID
- Outputs/Return: `true` if resource exists, `false` otherwise
- Side effects: None
- Calls: (implementation not visible)
- Notes: Lightweight existence check

**Trivial helper functions:** `close_res_file()`, `open_res_file_from_rwops()`, `cur_res_file()`, `get_1_resource_id_list()`, `get_resource_id_list()` ΓÇö manage file handles and retrieve ID lists.

## Control Flow Notes
This module is part of the resource loading and initialization phase. `initialize_resources()` is called early in engine startup. Resource files are opened (typically via `open_res_file()`) and set as current via `use_res_file()`. During game initialization, resources are enumerated and loaded using `count_*_resource()`, `get_*_resource_id_list()`, and `get_*_resource()`. Resources may be accessed throughout runtime for asset lookup.

## External Dependencies
- **stdio.h**: Standard C I/O (minimal usage in header)
- **vector (STL)**: Dynamic arrays for resource ID lists
- **SDL.h**: SDL library; provides SDL_RWops abstraction for file I/O
- **FileSpecifier** (defined elsewhere): File path abstraction
- **LoadedResource** (defined elsewhere): Resource data container

---

**Note:** The naming convention (`*_1_*` vs. `*_*`) suggests a Classic MacOS resource manager pattern: single-file functions search only the current file, while chain functions search across all open resource files in priority order.
