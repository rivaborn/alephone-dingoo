# tools/dumprsrcmap.cpp

## File Purpose
Standalone command-line utility that parses and prints a formatted dump of Macintosh resource map files. Supports AppleSingle, MacBinary, and raw resource fork formats. Used for development/debugging to inspect resource file contents.

## Core Responsibilities
- Accept file path from command-line arguments
- Open and parse resource files via SDL abstraction
- Enumerate all resource types in the file
- For each type, iterate and list all resource IDs with their data sizes
- Format and print resource map to stdout
- Handle resource lifecycle (load/unload) during enumeration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class | File path abstraction (from FileHandler_SDL.cpp) |
| `res_file_t` | struct | Internal resource file object with nested type_map_t and id_map_t for resource organization |
| `LoadedResource` | class | Wrapper for in-memory resource data (defined elsewhere) |
| `SDL_RWops` | struct | SDL file I/O abstraction for platform-independent file operations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `set_game_error` | function stub | global | Empty dummy function to avoid link errors |
| `local_data_dir, preferences_dir, saved_games_dir, recordings_dir` | DirectorySpecifier | global | Directory path globals; stubbed for linking |
| `data_search_path` | vector<DirectorySpecifier> | global | Search path vector; stubbed for linking |

## Key Functions / Methods

### csprintf
- Signature: `char *csprintf(char *buffer, const char *format, ...)`
- Purpose: Printf-style string formatter using variadic arguments
- Inputs: Destination buffer, format string, variable arguments
- Outputs/Return: Returns buffer pointer
- Side effects: Fills buffer with formatted string
- Calls: `va_start`, `vsprintf`, `va_end`
- Notes: Wrapper around vsprintf; caller responsible for buffer size

### main
- Signature: `int main(int argc, char **argv)`
- Purpose: Program entry point; drives resource enumeration and output
- Inputs: argc (argument count), argv (argument vector where argv[1] is file path)
- Outputs/Return: Prints resource map to stdout; returns 0 on success, 1 on failure
- Side effects: Opens SDL_RWops file handle; loads/unloads resources during iteration; accesses global `cur_res_file_t`
- Calls: `open_res_file`, `r->count_resources`, `get_resource_id_list`, `get_resource`, `rsrc.Unload()`
- Notes: 
  - Validates argc ΓëÑ 2; exits with usage message if insufficient arguments
  - Accesses `cur_res_file_t` iterator (global, set by resource_manager.cpp)
  - Prints resource type as 4-character MacOS code with hex value
  - Nested iteration: types ΓåÆ IDs within each type

## Control Flow Notes
Standalone utility with linear control flow (no frame loop or persistent state):  
1. Validate command-line arguments
2. Open resource file via `open_res_file`
3. Iterate type_map in `res_file_t::types`
4. For each type, call `get_resource_id_list` to retrieve IDs
5. Load each resource, print its ID and size, unload
6. Exit

## External Dependencies
- **resource_manager.cpp** (`#include "resource_manager.cpp"`): Provides `open_res_file()`, `get_resource_id_list()`, `get_resource()`, global `cur_res_file_t`, `res_file_t` struct with nested maps
- **FileHandler_SDL.cpp** (`#include "FileHandler_SDL.cpp"`): Provides `FileSpecifier` class, `LoadedResource` class
- **csalerts_sdl.cpp**, **Logging.cpp**, **XML_ElementParser.cpp**: Included for linking but not directly used in this file
- **SDL**: `SDL_RWops` (file I/O abstraction)
- **Standard C**: `<stdio.h>`, `<stdlib.h>`, `<stdarg.h>`
- **Standard C++**: `<vector>`

**Note:** This tool compiles multiple .cpp files as a unit (unusual but valid for standalone utilities). All dynamic allocation and resource enumeration logic is defined in the included files.
