# Source_Files/CSeries/csfiles.cpp

## File Purpose
Provides macOS Classic/Carbon filesystem utilities for locating game data files and the running application executable. Handles file specification resolution with fallback path searching and bundle-aware application discovery.

## Core Responsibilities
- Resolve file specifications (FSSpec) from resource-based filenames and search paths
- Locate the running application's own file specification in the filesystem
- Support both Classic Mac and bundled application directory structures
- Perform path construction and file search iteration

## Key Types / Data Structures
None defined in this file. Uses Carbon/Classic Mac types:
| Name | Kind | Purpose |
|------|------|---------|
| FSSpec | struct (Carbon) | File specification: volume reference, directory ID, filename |
| ProcessSerialNumber | struct (Carbon) | Current process identifier |
| ProcessInfoRec | struct (Carbon) | Process information record |

## Global / File-Static State
None

## Key Functions / Methods

### get_file_spec
- **Signature:** `OSErr get_file_spec(FSSpec *spec, short listid, short item, short pathsid)`
- **Purpose:** Locates a file by attempting direct resolution first, then iterating through a list of search paths.
- **Inputs:**
  - `spec`: Pointer to FSSpec to be filled with result
  - `listid`: Resource ID for list containing target filename
  - `item`: Index in the list for the filename to search
  - `pathsid`: Resource ID for list of search path prefixes
- **Outputs/Return:** OSErr (noErr on success, fnfErr if not found)
- **Side effects:** Modifies FSSpec structure; reads resources via `getpstr` / `countstr`
- **Calls:** `getpstr`, `countstr` (csstrings), `FSMakeFSSpec` (Carbon), `memcpy` (C stdlib)
- **Notes:** Uses Pascal string manipulation (length byte in `pathstr[0]`); concatenates paths by inserting filename into path string; returns on first match.

### get_my_fsspec
- **Signature:** `OSErr get_my_fsspec(FSSpec *spec)`
- **Purpose:** Retrieves the file specification of the running application executable.
- **Inputs:** `spec` ΓÇô Pointer to FSSpec to be filled
- **Outputs/Return:** OSErr (noErr on success)
- **Side effects:** Modifies FSSpec structure; navigates bundle hierarchy if `APPLICATION_IS_BUNDLED` is defined
- **Calls:** `GetProcessInformation` (Carbon), `FSMakeFSSpec` (Carbon)
- **Notes:** Uses `kCurrentProcess` for current process; conditionally backs out of .app bundle by constructing ":::" path (three parent directory levels) on bundled builds.

## Control Flow Notes
These are startup/initialization utilities called before main game loop to discover data files and executable location. Not part of frame/render cycle.

## External Dependencies
- **Carbon.h** (macOS Classic/Carbon APIs: FSSpec, ProcessSerialNumber, ProcessInfoRec, GetProcessInformation, FSMakeFSSpec, kCurrentProcess)
- **csstrings.h** (`getpstr`, `countstr`)
- **csfiles.h** (own declarations)
- **<string.h>** (memcpy)
