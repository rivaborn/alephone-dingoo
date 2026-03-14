# Source_Files/Files/wad_sdl.cpp

## File Purpose
SDL-based implementation for locating game map (WAD) files within configurable search paths. Provides two search strategies: by embedded checksum value or by modification date. Acts as a utility layer for the resource discovery pipeline.

## Core Responsibilities
- Locate WAD files matching a 32-bit checksum embedded at file offset 0x44
- Locate files matching a specific modification date
- Iterate through a global data search path (`data_search_path`) to find files
- Use template-style file finder pattern (FileFinder subclasses) to abstract search logic

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FindByChecksum` | class (FileFinder subclass) | Searches for files by comparing a 32-bit big-endian value at offset 0x44 to a target checksum |
| `FindByDate` | class (FileFinder subclass) | Searches for files by comparing their modification date to a target date |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | extern (defined in shell_sdl.cpp) | Ordered list of directories to search for game files |

## Key Functions / Methods

### find_wad_file_that_has_checksum
- Signature: `bool find_wad_file_that_has_checksum(FileSpecifier &matching_file, Typecode file_type, short path_resource_id, uint32 checksum)`
- Purpose: Locate a WAD file with a specific embedded checksum across all search paths
- Inputs: target checksum (uint32), file type typecode, output FileSpecifier reference
- Outputs/Return: true if found; `matching_file` contains the located file
- Side effects: Creates temporary FindByChecksum object; opens and reads files during search
- Calls: `FindByChecksum()` constructor, `finder.Find()`, file opening (via FileFinder)
- Notes: `path_resource_id` parameter is unused; searches are depth-first through `data_search_path` iterator

### find_file_with_modification_date
- Signature: `bool find_file_with_modification_date(FileSpecifier &matching_file, Typecode file_type, short path_resource_id, TimeType modification_date)`
- Purpose: Locate a file with a specific modification timestamp across all search paths
- Inputs: target modification date (TimeType/time_t), file type typecode, output FileSpecifier reference
- Outputs/Return: true if found; `matching_file` contains the located file
- Side effects: Creates temporary FindByDate object; calls FileSpecifier::GetDate() on candidates
- Calls: `FindByDate()` constructor, `finder.Find()`
- Notes: `path_resource_id` parameter is unused; parallel structure to checksum search

### FindByChecksum::found (nested)
- Purpose: Callback invoked by FileFinder for each discovered file; checks checksum match
- Inputs: FileSpecifier reference to candidate file
- Outputs/Return: true if file matches target checksum
- Side effects: Opens file, seeks to offset 0x44, reads 4 bytes
- Calls: `file.Open()`, `f.SetPosition()`, `f.GetRWops()`, `SDL_ReadBE32()`
- Notes: Offset 0x44 is a fixed position in Marathon/Aleph One WAD format; big-endian read via SDL

### FindByDate::found (nested)
- Purpose: Callback invoked by FileFinder for each discovered file; checks date match
- Inputs: FileSpecifier reference to candidate file
- Outputs/Return: true if file's modification date matches target
- Side effects: Calls file metadata query
- Calls: `file.GetDate()`

## Control Flow Notes
This module is called during resource discovery/loading phases (likely in initialization or when searching for game data). It does not participate directly in frame loops; instead, it supports higher-level game systems that need to locate map files or auxilary data files. The search is linear and stops at the first match.

## External Dependencies
- **SDL**: `SDL_endian.h` (SDL_ReadBE32 for endian-safe big-endian integer reads)
- **FileHandler.h**: FileSpecifier, OpenedFile, Typecode, TimeType, DirectorySpecifier abstractions
- **find_files.h**: FileFinder base class defining the callback-based enumeration pattern
- **cseries.h**: General utility types and platform macros
- **shell_sdl.cpp** (external): defines `data_search_path` vector
