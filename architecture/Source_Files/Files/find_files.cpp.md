# Source_Files/Files/find_files.cpp

## File Purpose
Implements the `FileFinder` class for recursively searching Mac OS directories for files matching a type criteria. Provides both buffer-based and callback-only search modes with support for directory change notifications.

## Core Responsibilities
- Clear FileFinder state (`Clear()`)
- Initiate recursive file search from a base directory (`Find()`)
- Recursively enumerate files and subdirectories in a directory tree (`Enumerate()`)
- Invoke file-matching callbacks and apply filtering based on type and flags
- Invoke directory entry/exit callbacks when recursing into subdirectories
- Support early termination via callback return values

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileFinder` | class | Main search object; encapsulates search parameters, results, and state |
| `DirectorySpecifier` | class (external) | Represents a directory location; passed by reference |
| `FileSpecifier` | class (external) | Represents a file; stored in results buffer or passed to callbacks |
| `CInfoPBRec` | struct (external) | Mac OS catalog info parameter block for `PBGetCatInfo` |

## Global / File-Static State
None. State is instance-based within `FileFinder` member variables.

## Key Functions / Methods

### Clear
- **Signature:** `void FileFinder::Clear()`
- **Purpose:** Initialize/reset all member variables to zero.
- **Inputs:** None (operates on `*this`).
- **Outputs/Return:** None.
- **Side effects:** Clears all search parameters, results, and internal state.
- **Calls:** `obj_clear(*this)` (utility macro/function).
- **Notes:** Called before reusing a FileFinder instance.

### Find
- **Signature:** `bool FileFinder::Find()`
- **Purpose:** Main entry point; initiates recursive file search from `BaseDir`.
- **Inputs:** FileFinder members: `Type`, `BaseDir`, `search_type`, `buffer`, `max`, `flags`, `callback`, `user_data`, `directory_change_callback`.
- **Outputs/Return:** `true` if search completed successfully (`Err == noErr`), `false` on error.
- **Side effects:** Populates `buffer` (if `_fill_buffer` mode) or invokes callbacks; increments `count`; sets `Err`.
- **Calls:** `get_typecode(Type)`, `Enumerate(BaseDir)`.
- **Notes:** Converts `Type` to a Mac OS typecode; asserts preconditions (`search_type==_callback_only || buffer`, `version==0`); resets `count` to 0.

### Enumerate
- **Signature:** `bool FileFinder::Enumerate(DirectorySpecifier& Dir)`
- **Purpose:** Recursively enumerate files/subdirectories in a given directory.
- **Inputs:** `Dir` (directory to enumerate); internally uses `search_type`, `Type`, `flags`, `callback`, `user_data`, `directory_change_callback`, `max`, `count`, `buffer`.
- **Outputs/Return:** `true` if enumeration succeeded or reached fnfErr (end-of-files), `false` on other errors.
- **Side effects:** Modifies `TempFile`, `pb`, `count`, `Err`, and `KeepGoing`; may recursively call itself; invokes callbacks.
- **Calls:** `TempFile.FromDirectory()`, `TempFile.GetSpec()`, `Dir.Get_vRefNum()`, `Dir.Get_parID()`, `TempFile.GetType()`, `PBGetCatInfo()`, directory/file callbacks.
- **Notes:**
  - Loops through directory entries using `PBGetCatInfo()` with incrementing index.
  - Checks `ioFlAttrib & 16` to detect subdirectories; recurses if `_ff_recurse` flag is set.
  - Invokes `directory_change_callback` on entering and exiting subdirectories (ZZZ addition).
  - Supports two search modes: `_fill_buffer` (appends matching files to buffer), `_callback_only` (invokes callback for each match).
  - Callback can return `false` to stop enumeration (especially in `_callback_only` mode).
  - Treats `fnfErr` (file not found, signals end of directory) as success.

## Control Flow Notes
- **Initialization:** External code sets FileFinder members, then calls `Find()`.
- **Frame:** `Find()` is a synchronous, blocking call that completes the entire search.
- **Recursion:** `Enumerate()` recursively traverses the directory tree depth-first if `_ff_recurse` is set, managing `TempFile` state across calls.
- **Termination:** Enumeration stops when `count >= max`, `Err != noErr`, `KeepGoing == false`, or all entries are exhausted.

## External Dependencies
- **Includes:** `"find_files.h"` (header), `"macintosh_cseries.h"` (Mac utilities), `<string.h>`, `<stdlib.h>`.
- **External symbols:**
  - `obj_clear()` ΓÇö memory utility macro/function.
  - `get_typecode()` ΓÇö converts type identifier to Mac OS typecode.
  - `PBGetCatInfo()` ΓÇö Mac OS API to retrieve directory/file catalog info.
  - `DirectorySpecifier`, `FileSpecifier`, `CInfoPBRec` ΓÇö defined elsewhere (FileHandler.h or Mac headers).
- **Conditional:** Code is only compiled on Mac (`#if defined(mac)`); SDL version exists in header as alternative class hierarchy.
