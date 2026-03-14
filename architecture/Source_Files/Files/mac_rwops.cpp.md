# Source_Files/Files/mac_rwops.cpp

## File Purpose
Provides SDL_RWops callback implementations optimized for reading from classic Mac OS resource and data forks. Acts as a bridge between SDL's cross-platform Read/Write abstraction and the Mac OS File Manager API.

## Core Responsibilities
- Implement SDL_RWops seek/read/write/close callbacks for Mac file handles
- Convert SDL seek constants (RW_SEEK_SET/CUR/END) to Mac OS File Manager equivalents (fsFromStart/fsFromMark/fsFromLEOF)
- Provide factory function to open Mac file forks and wrap them in SDL_RWops
- Support both resource fork and data fork access on classic Mac OS files
- Manage lifecycle of Mac file handles (open/close)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SDL_RWops | struct (external) | SDL abstraction for read/write operations; holds function pointers and opaque file handle data |
| FSSpec | struct (external) | Mac OS file specification structure for identifying files by path |

## Global / File-Static State
None.

## Key Functions / Methods

### rw_seek
- **Signature:** `static int SDLCALL rw_seek(SDL_RWops *context, int offset, int whence)`
- **Purpose:** Seek to a position in an open Mac file
- **Inputs:** SDL_RWops context, byte offset, whence (RW_SEEK_SET/CUR/END)
- **Outputs/Return:** New absolute file position; ΓêÆ1 on error
- **Side effects:** Calls Mac File Manager SetFPos/GetFPos; changes file position
- **Calls:** SetFPos, GetFPos
- **Notes:** Maps SDL constants to Mac OS File Manager constants; always verifies position after seek

### rw_read
- **Signature:** `static int SDLCALL rw_read(SDL_RWops *context, void *ptr, int size, int maxnum)`
- **Purpose:** Read data from a Mac file into buffer
- **Inputs:** SDL_RWops context, output buffer ptr, element size, max element count
- **Outputs/Return:** Number of elements successfully read; 0 on error
- **Side effects:** FSRead call; writes to output buffer
- **Calls:** FSRead
- **Notes:** Converts FSRead's byte count back to element count; total_bytes modified in-place by FSRead

### rw_write
- **Signature:** `static int SDLCALL rw_write(SDL_RWops *context, const void *ptr, int size, int num)`
- **Purpose:** Write callback (intentionally stubbed)
- **Inputs:** SDL_RWops context, data ptr, element size, count
- **Outputs/Return:** Implicitly 0 (no return statement)
- **Side effects:** None
- **Notes:** Stub with `assert(true)`; read-only wrapper, writes not supported

### rw_close
- **Signature:** `static int SDLCALL rw_close(SDL_RWops *context)`
- **Purpose:** Close a Mac file and deallocate SDL_RWops
- **Inputs:** SDL_RWops context
- **Outputs/Return:** 0 (always succeeds)
- **Side effects:** FSClose closes the file; SDL_FreeRW deallocates the RWops struct
- **Calls:** FSClose, SDL_FreeRW
- **Notes:** Safe null-check before operations

### open_fork_from_existing_path
- **Signature:** `SDL_RWops * open_fork_from_existing_path(const char *inPath, bool resFork)`
- **Purpose:** Factory function: open a Mac file fork and return a configured SDL_RWops for it
- **Inputs:** inPath (C-string file path), resFork (true = resource fork, false = data fork)
- **Outputs/Return:** Configured SDL_RWops pointer; NULL on any error
- **Side effects:** FSMakeFSSpec (Mac file lookup), FSpOpenRF/FSpOpenDF (opens file), SDL_AllocRW (allocates RWops)
- **Calls:** FSMakeFSSpec, FSpOpenRF or FSpOpenDF, SDL_AllocRW
- **Notes:** Converts C string to Mac OS Pascal string (length-prefixed); opens with read-only permission (fsRdPerm); stores file RefNum in RWops.hidden.unknown.data1

## Control Flow Notes
This file provides a utility layer activated during resource loading / initialization phases. The factory function opens a Mac fork and returns an opaque SDL_RWops that the caller uses without knowledge of platform-specific details. The four callback pointers (seek/read/write/close) are invoked by SDL code using the RWops abstraction.

## External Dependencies
- **SDL.h:** SDL_RWops, SDL_AllocRW, SDL_FreeRW, RW_SEEK_* constants
- **Files.h, Resources.h:** Mac OS File Manager APIs (FSSpec, FSMakeFSSpec, FSpOpenRF, FSpOpenDF, FSRead, FSClose, SetFPos, GetFPos)
- **cseries.h:** Project-wide definitions and includes

**Known issues (not inferable safety issues):**
- Operator precedence bug in `open_fork_from_existing_path`: the `!= noErr` check only applies to the FSpOpenDF branch, not FSpOpenRF
- Buffer overflow risk: strcpy used to build Pascal string without bounds checking
