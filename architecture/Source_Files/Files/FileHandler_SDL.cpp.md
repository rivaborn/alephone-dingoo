# Source_Files/Files/FileHandler_SDL.cpp

## File Purpose
SDL implementation of cross-platform file I/O for the Aleph One game engine. Provides abstractions for file operations, resource management, and file/directory selection dialogs with transparent handling of macOS resource forks and legacy file formats (AppleSingle, MacBinary).

## Core Responsibilities
- File I/O abstraction via SDL_RWops with position/length tracking and fork offset support
- Resource data lifecycle management (load, unload, detach)
- File path manipulation, type detection, directory operations across Windows/Unix/macOS
- Interactive file/directory browsing dialogs with sorting and navigation
- Automatic file type identification via magic bytes (Sounds, Maps/Scenarios, Physics, Shapes)
- File extension management and type code mapping
- Safe file operations (existence checks, overwrite confirmation, atomic exchange)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| OpenedFile | class | Wraps SDL_RWops with read/write/seek and fork offset management |
| LoadedResource | class | Manages malloc'd resource memory with unload cleanup |
| OpenedResourceFile | class | Resource file operations with context push/pop state management |
| FileSpecifier | class | File path abstraction with create/delete/open/type detection/dialog support |
| dir_entry | struct | Directory entry with name, size, modification date, type flags |
| w_directory_browsing_list | class | Interactive directory tree widget with sorting and up-button navigation |
| w_file_list | class | Base widget for rendering file list items with text clipping |
| w_read_file_list, w_write_file_list | class | Specialized file lists for open/save dialogs |
| w_file_name | class | Text entry field accepting Return to submit |
| extension_mapping | struct | File extension ΓåÆ typecode lookup entry |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| data_search_path | vector<DirectorySpecifier> | global (external) | Search path for game data files |
| local_data_dir, preferences_dir, saved_games_dir, recordings_dir | DirectorySpecifier | global (external) | User-specific directory locations |
| alephone_extensions[] | static const char* | file-static | Known Aleph One file extensions (.sceA, .sgaA, .filA, etc.) |
| extensions[] | static extension_mapping | file-static | ExtensionΓåÆtypecode mapping table |
| temporary | static char[256] | file-static | Temporary path/name buffer (implicit from code) |

## Key Functions / Methods

### FileSpecifier::Open (OpenedFile overload)
- **Signature:** `bool Open(OpenedFile &OFile, bool Writable)`
- **Purpose:** Open file for reading or writing with transparent resource fork handling
- **Inputs:** Target OpenedFile object, Writable flag
- **Outputs/Return:** true if opened successfully; OFile.f set to SDL_RWops handle; error code in OFile.err
- **Side effects:** Opens SDL_RWops; checks AppleSingle/MacBinary headers on read; seeks to data fork offset
- **Calls:** `SDL_RWFromFile()`, `is_applesingle()`, `is_macbinary()`, `SDL_RWseek()`
- **Notes:** On macOS, uses special fork-opening function. Transparently handles forked format offsets.

### FileSpecifier::GetType
- **Signature:** `Typecode GetType()`
- **Purpose:** Detect file type by examining extension and/or magic bytes
- **Inputs:** (implicit) file path in `name`
- **Outputs/Return:** Typecode enum (_typecode_sounds, _typecode_scenario, _typecode_shapes, etc.)
- **Side effects:** Opens file temporarily; scans header bytes; closes file if needed
- **Calls:** `Open()`, `SDL_ReadBE32()`, `SDL_ReadBE16()`, `strcasecmp()`
- **Notes:** Extension-based detection fast-tracks common types. Magic byte checks validate maps/sounds/shapes. Returns _typecode_unknown as fallback.

### FileSpecifier::ReadDialog
- **Signature:** `bool ReadDialog(Typecode type, const char *prompt)`
- **Purpose:** Show interactive file browser for opening a file
- **Inputs:** File type (affects directory and sorting defaults); custom prompt text
- **Outputs/Return:** true if user selected file; FileSpecifier set to selected file path
- **Side effects:** Creates modal dialog; manages w_directory_browsing_list widget; redraws game window
- **Calls:** Dialog widget constructors, `refresh_entries()`, game window redraw
- **Notes:** Type-specific defaults (savegame sorted by date; scenario/film use standard dirs). Handles netscript starting file.

### FileSpecifier::WriteDialog
- **Signature:** `bool WriteDialog(Typecode type, const char *prompt, const char *default_name)`
- **Purpose:** Show interactive file save dialog with overwrite confirmation
- **Inputs:** File type, prompt text, default filename
- **Outputs/Return:** true if user confirmed save; FileSpecifier set to target path with extension appended
- **Side effects:** Creates dialog; appends extension (.sgaA, .filA) if not present; calls confirm_save_choice()
- **Calls:** `confirm_save_choice()`, dialog widget constructors
- **Notes:** Loops if user enters empty name or on overwrite cancel. Extension auto-append per typecode.

### FileSpecifier::ReadDirectory
- **Signature:** `bool ReadDirectory(vector<dir_entry> &vec)`
- **Purpose:** List contents of directory
- **Inputs:** Target vector for results
- **Outputs/Return:** true on success; vec populated with sorted dir_entry objects
- **Side effects:** Reads filesystem; filters dot files and .. directory
- **Calls:** Platform-specific: `opendir()`/`readdir()`/`closedir()` (POSIX) or `FindFirstFile()`/`FindNextFile()` (Win32)
- **Notes:** Excludes hidden files (starting with '.') except as directory markers. Sorts with dirs before files.

### FileSpecifier::CopyContents
- **Signature:** `bool CopyContents(FileSpecifier &source_name)`
- **Purpose:** Copy entire file contents to this file location
- **Inputs:** Source FileSpecifier
- **Outputs/Return:** true on success; writes data in 1024-byte chunks
- **Side effects:** Deletes target file; opens source and destination; copies data; deletes target on error
- **Calls:** `Open()`, `Delete()`, `Read()`, `Write()`, `GetLength()`

## Control Flow Notes
File operations integrate into game lifecycle: data loading (maps, sounds, shapes, physics) on startup; save/load dialogs during gameplay (pause menu); file dialogs for preferences/recordings. Directory browsing dialogs are modalΓÇöthey block input until user navigates and selects or cancels. Resource file Push/Pop maintains global resource context for proper resource fork switching on macOS.

## External Dependencies
- **SDL:** `SDL_RWops`, `SDL_RWFromFile()`, `SDL_RWread/write/seek/tell/close()`, endian conversion functions
- **POSIX/Win32:** `stat()`, `mkdir()`, `access()`, `opendir/readdir/closedir()`, `rename()`, `remove()` (POSIX); `FindFirstFile/NextFile`, `GetLastError()` (Win32)
- **Engine:** `resource_manager.h` (resource fork operations), `shell.h` (directory specifiers), `game_errors.h` (error reporting), `SoundManager.h`, preferences system
- **UI:** `sdl_dialogs.h`, `sdl_widgets.h` (dialog/widget framework)
- **Other:** Boost string algorithms (`ends_with`), tags.h (typecode constants)
