# Expat/MacOS Support/OpenSave_Interface.h

## File Purpose
Defines a cross-platform interface for file open/save dialogs, abstracting platform differences between Win32 and MacOS. Provides parameter structs and two entry-point functions to invoke native dialog boxes and retrieve the user's file selection as a pathname string.

## Core Responsibilities
- Define parameter structures for open and save file dialogs
- Encapsulate platform-specific options (Win32 suffixes, MacOS type/creator codes) in a single interface
- Declare `OpenFile()` and `SaveFile()` functions for invoking platform dialogs
- Document memory management expectations (malloc/free for returned strings)
- Provide guidance on safe file operations (temporary file pattern for saves)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OpenParameters` | struct | Bundles options for file open dialogs: file suffix (Win32), type codes (MacOS), and prompt text |
| `SaveParameters` | struct | Bundles options for file save dialogs: suffix (Win32), type/creator codes (MacOS), prompt, and default filename |

## Global / File-Static State
None.

## Key Functions / Methods

### OpenFile
- **Signature:** `extern char *OpenFile(struct OpenParameters *OpenParms);`
- **Purpose:** Invoke a native file open dialog and return the user-selected file path.
- **Inputs:** Pointer to `OpenParameters` struct (suffix/types, prompt).
- **Outputs/Return:** Dynamically allocated C string containing full pathname; caller must free with `free()`.
- **Side effects:** Likely blocks on user interaction; does not open the file itself.
- **Calls:** Not visible (platform-specific implementation, likely in separate Win32 or MacOS translation units).
- **Notes:** Returns NULL or empty string if user cancels. String allocated with `malloc()`, not `new`.

### SaveFile
- **Signature:** `extern char *SaveFile(struct SaveParameters *SaveParms);`
- **Purpose:** Invoke a native file save dialog and create the file at the user-selected path.
- **Inputs:** Pointer to `SaveParameters` struct (suffix/type codes, prompt, default name).
- **Outputs/Return:** Dynamically allocated C string containing full pathname of created file; caller must free with `free()`.
- **Side effects:** Creates file at selected path; replaces any existing file with that name (unsafe). Blocks on user interaction.
- **Calls:** Not visible (platform-specific implementation).
- **Notes:** Comments suggest a safer pattern using temporary files and atomic swap, but not implemented here.

## Control Flow Notes
Not inferable from this file. This is a pure interface header; actual dialog invocation and platform integration occurs in separate translation units (likely `OpenSave_Interface_Win32.c` and `OpenSave_Interface_MacOS.c` or equivalent).

## External Dependencies
- C standard types: `char`, `short`, `unsigned long`.
- C++ extern "C" guards to allow C++ callers to link with C implementations.
- No explicit includes visible.
