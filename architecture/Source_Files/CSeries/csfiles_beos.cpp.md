# Source_Files/CSeries/csfiles_beos.cpp

## File Purpose
BeOS-specific utility file for the Aleph One game engine. Provides directory discovery for application and preferences locations, and implements SDL-compatible I/O abstractions for reading/writing resource forks stored as BeOS file system attributesΓÇöenabling Marathon data files from Mac CD-ROMs to be used without conversion.

## Core Responsibilities
- Locate application and user preferences directories on BeOS via AppKit/StorageKit
- Detect resource fork attributes on files using BeOS filesystem APIs
- Wrap resource fork data in an SDL_RWops interface for transparent read/write access
- Manage file handle lifecycle and seek/read/write position tracking for resource fork streams

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rfork_data` | struct | Encapsulates open file descriptor, current stream position, and total attribute size for resource fork I/O |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ATTR_NAME` | `#define` (string literal) | file-static | The BeOS attribute name identifying resource forks: `"MACOS:RFORK"` |

## Key Functions / Methods

### get_application_directory
- **Signature:** `string get_application_directory(void)`
- **Purpose:** Locate the directory containing the running application executable.
- **Inputs:** None.
- **Outputs/Return:** Absolute filesystem path as `string`.
- **Side effects:** None.
- **Calls:** `be_app->GetAppInfo()`, `BEntry()`, `BPath()`, `path.GetParent()`.
- **Notes:** Uses BeOS AppKit to query app info; navigates one level up from executable.

### get_preferences_directory
- **Signature:** `string get_preferences_directory(void)`
- **Purpose:** Locate the user's preferences directory for Aleph One configuration.
- **Inputs:** None.
- **Outputs/Return:** Absolute filesystem path as `string`.
- **Side effects:** Creates the `"Aleph One"` subdirectory in `B_USER_SETTINGS_DIRECTORY` if missing (flag `true`).
- **Calls:** `find_directory()`, `prefs_dir.Append()`.
- **Notes:** Delegates to BeOS StorageKit; standardizes config storage in system user settings.

### has_rfork_attribute
- **Signature:** `bool has_rfork_attribute(const char *file)`
- **Purpose:** Check whether a file has a resource fork attribute.
- **Inputs:** `file` ΓÇô filesystem path (C string).
- **Outputs/Return:** `true` if attribute exists, `false` otherwise.
- **Side effects:** Opens/closes file descriptor.
- **Calls:** `open()`, `fs_stat_attr()`, `close()`.
- **Notes:** Returns `false` on open failure; uses read-only mode.

### sdl_rw_from_rfork
- **Signature:** `SDL_RWops *sdl_rw_from_rfork(const char *file, bool writable)`
- **Purpose:** Factory function: create an SDL_RWops interface for reading/writing a file's resource fork attribute.
- **Inputs:** `file` ΓÇô filesystem path; `writable` ΓÇô read-write (true) or read-only (false) mode.
- **Outputs/Return:** Pointer to initialized `SDL_RWops` structure, or `NULL` on error.
- **Side effects:** Allocates heap memory for `SDL_RWops` and `rfork_data`; opens file descriptor; sets SDL error state on failure.
- **Calls:** `open()`, `fs_stat_attr()`, `SDL_AllocRW()`, `malloc()`, `SDL_SetError()`, `close()`.
- **Notes:** Caller must manage returned pointer; file must remain open for lifetime of SDL_RWops. On allocation failure, properly cleans up file descriptor and partially allocated structures.

### rfork_seek, rfork_read, rfork_write, rfork_close
- **Purpose:** SDL_RWops callback implementations (seek, read, write, close operations).
- **Inputs:** `context` (SDL_RWops*), operation-specific parameters.
- **Outputs/Return:** Standard SDL return values (position for seek, bytes read/written for I/O, 0 for close).
- **Side effects:** Modify `d->current` position; I/O via `fs_read_attr()` / `fs_write_attr()`; close/free on close callback.
- **Calls:** `SDL_SetError()`, `SDL_Error()`, `fs_read_attr()`, `fs_write_attr()`, `close()`, `free()`.
- **Notes:** Bounds-check seeks; clamp read/write to remaining attribute size; `rfork_close` frees both file descriptor context and SDL structure.

## Control Flow Notes
This file is initialization/utility-only, not part of the main game loop. Functions are called during:
- **Startup:** Directory discovery for config/resource loading.
- **File I/O:** On-demand creation of resource fork streams when loading data from Mac-format media.

## External Dependencies
- **SDL:** `SDL_rwops.h`, `SDL_error.h` ΓÇô RWops abstraction and error handling.
- **BeOS AppKit:** `<AppKit.h>` ΓÇô `be_app`, `BEntry`, `BPath`, `app_info`.
- **BeOS StorageKit:** `<StorageKit.h>` ΓÇô `find_directory()`, path management.
- **POSIX:** `<unistd.h>`, `<fcntl.h>` ΓÇô file operations (`open`, `close`).
- **BeOS FS API:** `<fs_attr.h>` ΓÇô attribute stat/read/write (`fs_stat_attr`, `fs_read_attr`, `fs_write_attr`).
- **Standard:** `<string>` ΓÇô C++ string type.
