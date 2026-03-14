# Expat/MacOS Support/GetFullPathObject.h

## File Purpose
A utility struct that encapsulates the logic for resolving the full filesystem path of a macOS FSSpec object. Acts as a simple function object wrapper, storing both the resolution result and any errors that occur during the operation.

## Core Responsibilities
- Provides a method to retrieve the full filesystem path from a macOS FSSpec
- Maintains the resolved path as a C string in a custom vector container
- Stores macOS error codes from the path resolution operation
- Initializes error state on construction

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GetFullPathObject` | struct | Function object wrapper for path resolution |
| `simple_vector<char>` | template class | Dynamic C string storage (length excludes null terminator) |
| `FSSpec` | (Carbon.h) | macOS filesystem specification type |
| `OSErr` | (Carbon.h) | macOS error code type |

## Global / File-Static State
None.

## Key Functions / Methods

### GetFullPath
- **Signature:** `bool GetFullPath(FSSpec *Spec);`
- **Purpose:** Resolves the full filesystem path for a macOS FSSpec object.
- **Inputs:** Pointer to `FSSpec` containing the file system object specification.
- **Outputs/Return:** Boolean success/failure; actual path stored in `FullPath` member.
- **Side effects:** Populates `FullPath` vector; sets `Err` field on error.
- **Calls:** Implementation not visible in this file.
- **Notes:** Returns bool for success, while `Err` stores macOS error codes. Per comment, `Err` is set to `noErr` on success or for allocation errors.

### Constructor
- **Signature:** `GetFullPathObject(): Err(noErr) {}`
- **Purpose:** Initialize with default state.
- **Inputs:** None.
- **Outputs/Return:** Initialized `GetFullPathObject` instance.
- **Side effects:** Sets `Err` to `noErr`; `FullPath` default-constructed (empty).

## Member Variables
- `OSErr Err` ΓÇô macOS error code (initialized to `noErr`)
- `simple_vector<char> FullPath` ΓÇô resolved path as C string

## Control Flow Notes
Not inferable from this file. Likely used during file system initialization or path resolution phases (not frame-bound).

## External Dependencies
- `<Carbon.h>` ΓÇô macOS Carbon framework (FSSpec, OSErr)
- `"SimpleVec.h"` ΓÇô Custom dynamic vector template
- `GetFullPath()` implementation defined elsewhere
