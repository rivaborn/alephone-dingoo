# Expat/MacOS Support/MoreFilesExtract.h

## File Purpose
A C header file declaring the `FSpGetFullPath` function extracted from the MoreFiles macOS filesystem library. This provides a wrapper for resolving full pathnames from FSSpec records on classic macOS.

## Core Responsibilities
- Declare the `FSpGetFullPath` function interface
- Document parameter semantics and memory ownership
- Enumerate error codes for filesystem resolution failures

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FSSpec | struct (from Carbon.h) | Specifies a volume, directory, or file on macOS |
| OSErr | typedef (from Carbon.h) | macOS error code type |
| Handle | typedef (from Carbon.h) | macOS memory handle (opaque reference) |

## Global / File-Static State
None.

## Key Functions / Methods

### FSpGetFullPath
- **Signature:** `OSErr FSpGetFullPath(const FSSpec *spec, short *fullPathLength, Handle *fullPath)`
- **Purpose:** Resolve a full absolute pathname for a volume, directory, or file specified by FSSpec.
- **Inputs:**
  - `spec`: pointer to FSSpec record identifying the target object
- **Outputs/Return:**
  - Returns `OSErr` (0 = success, negative = error code)
  - `fullPathLength`: receives character count of the pathname; set to 0 on failure
  - `fullPath`: receives a newly allocated Handle to the pathname buffer; set to NULL on failure
- **Side effects:** Allocates memory; caller must dispose of `fullPath` handle via macOS memory manager
- **Calls:** Implementation defined elsewhere (from MoreFiles library)
- **Notes:**
  - Caller owns memory allocation responsibility
  - Supports multiple error conditions: volume not found, I/O errors, bad filenames, directory not found, access denied, insufficient memory
  - Extracted/wrapped from MoreFiles library; original implementation not visible in this header

## Control Flow Notes
This is a pure interface declaration. `FSpGetFullPath` would be called during filesystem traversal or path resolution tasks to convert opaque FSSpec identifiers into human-readable pathnames.

## External Dependencies
- **Carbon.h:** Provides `FSSpec`, `OSErr`, `Handle`, and macOS Toolbox filesystem types
- **MoreFiles library:** Original implementation of `FSpGetFullPath` (not defined in this file)
