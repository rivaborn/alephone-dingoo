# Source_Files/Files/find_files.h

## File Purpose
Provides cross-platform file-finding abstractions for both macOS (File Manager API) and SDL/generic platforms. Enables recursive directory traversal with type-based filtering and callback support for file discovery operations.

## Core Responsibilities
- Abstract file discovery across macOS FSSpec and SDL/cross-platform filesystem APIs
- Search directories for files matching a specified Typecode (file type)
- Support recursive traversal with subdirectory enumeration
- Buffer management for search results (count and destination array)
- Callback invocation for file filtering and directory change notifications
- Platform-specific error reporting

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FileFinder (mac) | class | macOS-specific file finder using FSSpec and CInfoPBRec for directory/file enumeration |
| FileFinder (SDL) | class (base) | Cross-platform base class with virtual `found()` callback interface |
| FindAllFiles | class | Concrete FileFinder subclass that collects results into a `vector<FileSpecifier>` |
| Typecode | typedef (from tags.h) | File type identifier; `_typecode_unknown` acts as wildcard |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| WILDCARD_TYPE | const Typecode | global | Constant set to `_typecode_unknown` for matching any file type |

## Key Functions / Methods

### FileFinder::Find() [macOS]
- Signature: `bool Find()`
- Purpose: Initiates the recursive file search based on current state (BaseDir, Type, flags)
- Inputs: Implicit state (BaseDir, Type, flags, callback, user_data, buffer, max)
- Outputs/Return: `true` if search completed (count indicates matches found)
- Side effects: Populates buffer with FileSpecifier matches; calls callback for each match; may invoke directory_change_callback
- Calls: `Enumerate()`
- Notes: Respects `_ff_recurse` flag; stops if callback returns false

### FileFinder::Enumerate() [macOS]
- Signature: `bool Enumerate(DirectorySpecifier& Dir)`
- Purpose: Private recursive helper that enumerates a single directory and subdirectories
- Inputs: DirectorySpecifier reference
- Outputs/Return: `true` if enumeration completed
- Side effects: Updates CInfoPBRec state; invokes callbacks; populates buffer
- Calls: Implicit macOS File Manager APIs (via pb member)
- Notes: Recursive if `_ff_recurse` flag set; handles subdirectory change callbacks

### FileFinder::Clear() [macOS]
- Signature: `void Clear()`
- Purpose: Resets internal state and buffers
- Inputs: None
- Outputs/Return: None
- Side effects: Clears temporary file state

### FileFinder::Find() [SDL]
- Signature: `bool Find(DirectorySpecifier &dir, Typecode type, bool recursive = true)`
- Purpose: Searches a directory tree for files of a given type
- Inputs: target directory, file type code, recursion flag
- Outputs/Return: `true` if search succeeded
- Side effects: Invokes virtual `found()` method for each match; no buffer management
- Calls: `found()` (pure virtual, implemented by subclasses)
- Notes: Callback-driven architecture; different from macOS design

### FindAllFiles::found() [SDL]
- Signature: `bool found(FileSpecifier &file)`
- Purpose: Concrete callback implementation that accumulates results
- Inputs: FileSpecifier reference for matched file
- Outputs/Return: `false` (never aborts enumeration)
- Side effects: Appends file to internal vector
- Calls: vector::push_back()

### FindAllFiles::FindAllFiles() [SDL]
- Signature: `FindAllFiles(vector<FileSpecifier> &v)`
- Purpose: Constructor that stores reference to destination vector and clears it
- Inputs: Non-const reference to vector to populate
- Side effects: Clears the passed-in vector

## Control Flow Notes
**macOS**: Driven by `Find()` ΓåÆ `Enumerate()` ΓåÆ recursive traversal with File Manager PB calls. Results either buffered (if callback is null, add all; if returns true) or callback-only mode. Directory change callbacks fire on entry/exit.

**SDL**: Driven by `Find()` ΓåÆ virtual `found()` dispatch for each discovery. `FindAllFiles` subclass implements `found()` to populate a vector. Callback can abort enumeration (return false).

## External Dependencies
- **FileHandler.h**: Provides `FileSpecifier`, `DirectorySpecifier`, `Typecode` abstractions, and OpenedFile/OpenedResourceFile for file I/O
- **tags.h**: Defines typecode symbolic constants (`_typecode_unknown`, `_typecode_creator`, etc.)
- **macOS APIs** (conditional): `Files.h`, `Resources.h` ΓåÆ FSSpec, CInfoPBRec, OSErr, OSType, DirectoryID
- **SDL** (conditional): `<vector>` for result aggregation; canonicalized path handling
- **Standard C++**: `<vector>` for cross-platform result storage
