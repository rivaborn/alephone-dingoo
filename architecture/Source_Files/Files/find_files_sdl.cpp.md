# Source_Files/Files/find_files_sdl.cpp

## File Purpose
Implements SDL-based file discovery and enumeration for the Aleph One game engine. Provides recursive directory traversal with file type matching and abstract callback mechanisms for cross-platform asset location.

## Core Responsibilities
- Recursive directory traversal with optional type-code filtering
- Virtual callback pattern for flexible file discovery (abort-on-match or batch collection)
- Directory entry sorting (directories before files, then alphabetical)
- SDL platform abstraction for file system access

## Key Types / Data Structures
| Name | Kind | Purpose |
| ---- | ---- | ------- |
| FileFinder | class | Abstract base for file discovery implementations |
| FindAllFiles | class | Concrete subclass that collects all matching files into a vector |
| dir_entry | struct | Represents a single directory or file entry with metadata |

## Global / File-Static State
None.

## Key Functions / Methods

### FileFinder::Find
- **Signature:** `bool Find(DirectorySpecifier &dir, Typecode type, bool recursive)`
- **Purpose:** Primary entry point; recursively searches directory tree for files matching a type code.
- **Inputs:** `dir` (starting directory), `type` (file type filter, or `WILDCARD_TYPE` for any), `recursive` (enable subdirectory recursion)
- **Outputs/Return:** `bool` ΓÇö returns `true` if search should abort (when `found()` returns true), `false` to continue
- **Side effects:** Reads filesystem via `DirectorySpecifier::ReadDirectory()`; invokes virtual `found()` method for each matching file
- **Calls:** `DirectorySpecifier::ReadDirectory()`, `std::sort()` (on `dir_entry` objects), `FileSpecifier::GetType()`, recursive `Find()`, virtual `found()`
- **Notes:**
  - Directory entries are sorted using `dir_entry::operator<`, which orders directories before files, then alphabetically
  - Search terminates early if any `found()` call returns `true`
  - `WILDCARD_TYPE` (_typecode_unknown) matches any file type
  - Handles both regular files and directories; recurses into subdirectories if flag is set

### FindAllFiles::found
- **Signature:** `bool found(FileSpecifier &file)`
- **Purpose:** Callback invoked by `Find()` for each matched file; appends to destination vector.
- **Inputs:** `file` (matched FileSpecifier)
- **Outputs/Return:** `bool` ΓÇö always `false` (never abort search)
- **Side effects:** Appends `file` to `dest_vector` (passed to constructor)
- **Calls:** `std::vector::push_back()`
- **Notes:** Simple collector pattern; relies on `Find()` calling this for all matches

## Control Flow Notes
This code executes during **engine initialization/startup** when asset discovery is required. `FileFinder::Find()` is the main entry pointΓÇöit drives recursive traversal. Subclasses override `found()` to control what happens at each match (e.g., `FindAllFiles` collects all matches; custom subclasses might abort early or apply filtering). The design decouples search logic from result handling.

## External Dependencies
- **Includes:** `<vector>`, `<algorithm>` (for `std::sort`)
- **From FileHandler.h:** `DirectorySpecifier`, `FileSpecifier`, `Typecode`, `dir_entry`
- **From find_files.h:** `FileFinder` base class definition, `WILDCARD_TYPE` constant
- **Note:** Compiled only when `SDL_RFORK_HACK` is not defined (i.e., on SDL platforms, not native Mac with resource forks)
