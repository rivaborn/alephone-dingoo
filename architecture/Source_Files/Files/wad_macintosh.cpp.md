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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `find_checksum_private_data` | struct | Private callback data; holds target checksum for matching |
| `find_files_private_data` | struct | Private callback data for file enumeration; holds base file reference and parent checksum (currently unused/disabled code) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `target_modification_date` | `TimeType` | file-static | Stores target modification date for callback matching during directory enumeration |

## Key Functions / Methods

### find_wad_file_that_has_checksum
- **Signature:** `bool find_wad_file_that_has_checksum(FileSpecifier& MatchingFile, Typecode file_type, short path_resource_id, uint32 checksum)`
- **Purpose:** Public API to locate a single WAD file with a matching checksum by searching the app's root directory recursively
- **Inputs:** Target checksum value; typecode for file filtering; output file reference; legacy `path_resource_id` parameter (unused)
- **Outputs/Return:** Boolean success; found file written to `MatchingFile` reference
- **Side effects:** Modifies `MatchingFile` if match found; triggers recursive directory enumeration
- **Calls:** `Files_GetRootDirectory()`, `find_wad_file_with_checksum_in_directory()`
- **Notes:** Per comment, scoped to app directory only (LP change); `path_resource_id` is legacy and ignored

### find_file_with_modification_date
- **Signature:** `bool find_file_with_modification_date(FileSpecifier& MatchingFile, Typecode file_type, short path_resource_id, TimeType modification_date)`
- **Purpose:** Public API to locate a file matching a specific modification date
- **Inputs:** Target modification date; typecode filter; output file reference; legacy path parameter
- **Outputs/Return:** Boolean success; matching file written to `MatchingFile`
- **Side effects:** Updates global `target_modification_date`; modifies `MatchingFile`; recursive search
- **Calls:** `Files_GetRootDirectory()`, `find_file_with_modification_date_in_directory()`
- **Notes:** Parallel API to checksum search

### find_wad_file_with_checksum_in_directory
- **Signature:** `static bool find_wad_file_with_checksum_in_directory(FileSpecifier& MatchingFile, DirectorySpecifier& BaseDir, Typecode file_type, uint32 checksum)`
- **Purpose:** Helper implementing actual checksum search using `FileFinder` object with callback
- **Inputs:** Base directory to search; typecode filter; target checksum; output buffer
- **Outputs/Return:** Boolean success; first matching file in `MatchingFile`
- **Side effects:** Output file; potential `dprintf()` error logging; recursive directory traversal
- **Calls:** `FileFinder::Clear()`, `FileFinder::Find()`, `match_wad_checksum_callback()` (installed as callback), `wad_file_has_checksum()` (indirectly via callback)
- **Notes:** Sets `_ff_recurse` flag; limits result count to 1; includes error checking

### find_file_with_modification_date_in_directory
- **Signature:** `static bool find_file_with_modification_date_in_directory(FileSpecifier& MatchingFile, DirectorySpecifier& BaseDir, Typecode file_type, TimeType modification_date)`
- **Purpose:** Helper implementing actual modification-date search using `FileFinder` with callback
- **Inputs:** Base directory; typecode filter; target date; output buffer
- **Outputs/Return:** Boolean success; matching file in `MatchingFile`
- **Side effects:** Writes to global `target_modification_date` (parameter passing hack); output file; error logging
- **Calls:** `FileFinder::Clear()`, `FileFinder::Find()`, `match_modification_date_callback()`, `FileSpecifier::GetDate()`
- **Notes:** Sets `_ff_callback_with_catinfo` flag for catalog info; also limits to 1 match

### match_wad_checksum_callback
- **Signature:** `static bool match_wad_checksum_callback(FileSpecifier& File, void *data)`
- **Purpose:** Callback invoked per-file during checksum enumeration; filters by checksum match
- **Inputs:** Current file; opaque data pointer (cast to `find_checksum_private_data*`)
- **Outputs/Return:** Boolean indicating whether to include file in results
- **Side effects:** None (read-only)
- **Calls:** `wad_file_has_checksum()` (defined elsewhere in wad.h)
- **Notes:** Dereferences private data to access `checksum_to_match`

### match_modification_date_callback
- **Signature:** `static bool match_modification_date_callback(FileSpecifier& File, void *data)`
- **Purpose:** Callback invoked per-file during date enumeration; filters by modification date
- **Inputs:** Current file; unused data pointer
- **Outputs/Return:** Boolean indicating match
- **Side effects:** None (read-only)
- **Calls:** `FileSpecifier::GetDate()`
- **Notes:** Uses global `target_modification_date` variable instead of data parameter (design smell; date should be passed in data struct)

## Control Flow Notes

This is a utility module invoked on-demand during runtime resource loading. No frame/init/shutdown involvement visible. Typical flow:

1. Engine needs to locate a WAD file by checksum (e.g., loading a saved game) or by date
2. Public entry point (`find_wad_file_that_has_checksum` or `find_file_with_modification_date`) called with search criteria
3. Function retrieves app's root directory via `Files_GetRootDirectory()`
4. Helper function (`find_wad_file_with_checksum_in_directory` or `find_file_with_modification_date_in_directory`) sets up and executes `FileFinder` with recursive flags and callback installed
5. `FileFinder` recursively enumerates directories, invoking callback for each file
6. Callback filters files; first match returns `true` to stop enumeration
7. Matching `FileSpecifier` returned to caller

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
