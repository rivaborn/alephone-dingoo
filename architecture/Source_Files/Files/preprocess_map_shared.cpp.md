# Source_Files/Files/preprocess_map_shared.cpp

## File Purpose
Implements automatic game saving for Aleph One with intelligent filename generation. Provides functionality to save the current game state without user interaction, either overwriting the most recent save or creating a new uniquely-named file based on the current level name.

## Core Responsibilities
- Generate filesystem-safe filenames from level names (alphanumeric + spaces only)
- Create non-conflicting filename variants by appending numeric suffixes to prevent overwrites
- Execute automatic game saves with option to overwrite or create new files
- Report save results to the user via on-screen messages
- Handle platform-specific filename length constraints (Mac: 31 chars, others: 255 chars)

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kMaxFilenameChars` | `enum` constant | static | Platform-dependent maximum filename length |

## Key Functions / Methods

### strncpy_filename_friendly
- **Signature**: `static int32 strncpy_filename_friendly(char* outString, const char* inString, int32 inOutputStringBufferSize)`
- **Purpose**: Copy input string to output buffer, keeping only alphanumeric characters and spaces (filesystem-safe sanitization)
- **Inputs**: 
  - `outString`: destination buffer
  - `inString`: source string to filter
  - `inOutputStringBufferSize`: size of destination buffer
- **Outputs/Return**: strlen() of the resulting output string
- **Side effects**: Writes sanitized string to outString; zero-fills remaining buffer capacity
- **Calls**: `isalnum()`, `memset()`
- **Notes**: Assumes valid buffer sizes (asserted); processes character-by-character, advancing output pointer only for alphanumeric/space characters

### make_nonconflicting_filename_variant
- **Signature**: `static bool make_nonconflicting_filename_variant(FileSpecifier& inBaseName, FileSpecifier& outNonconflictingName, size_t inMaxNameLength)`
- **Purpose**: Generate a unique filename by iteratively appending numeric suffixes (" 2", " 3", etc.) until an unused filename is found
- **Inputs**: 
  - `inBaseName`: base filename (directory + name)
  - `inMaxNameLength`: maximum character length allowed in final name
- **Outputs/Return**: True (always succeeds; infinite loop risk noted as "really, REALLY tiny")
- **Side effects**: Calls `Exists()` on each variant until collision-free name found; modifies `outNonconflictingName`
- **Calls**: FileSpecifier methods (`ToDirectory`, `GetName`, `FromDirectory`, `SetName`/`AddPart`, `Exists`), `sprintf()`, `strlen()`
- **Notes**: 
  - First attempt has no suffix (variant 1); increments to " 2", " 3", etc.
  - Truncates basename if `basename_length + suffix_length > inMaxNameLength`
  - Platform-specific: Mac/SDL_RFORK_HACK use `SetName()`; others use `AddPart()`

### save_game_full_auto
- **Signature**: `bool save_game_full_auto(bool inOverwriteRecent)`
- **Purpose**: Automatically save the current game without presenting a dialog. Optionally overwrites the most recent save or generates a new uniquely-named file.
- **Inputs**: `inOverwriteRecent` ΓÇö if true, overwrite recent save (if valid); otherwise create new file
- **Outputs/Return**: True if save succeeded, false if save_game_file() failed
- **Side effects**: 
  - I/O: writes save file to disk
  - Calls `screen_printf()` to display status message ("Created newΓÇª" or "Replaced existingΓÇª" or error)
  - Comments indicate sound effects were planned but not implemented
- **Calls**: 
  - `get_current_saved_game_name()` ΓÇö retrieves recent save location
  - `TS_GetCString()` ΓÇö fetches default save name string resource
  - `strncpy_filename_friendly()` ΓÇö sanitizes level name for filename
  - `make_nonconflicting_filename_variant()` ΓÇö generates unique filename if creating new file
  - `save_game_file()` ΓÇö performs actual game state serialization
  - `screen_printf()` ΓÇö displays result to user
- **Notes**: 
  - Uses `static_world->level_name` (current map level) as base for new filename; falls back to "Automatic Save" if level name sanitizes to empty
  - Respects platform filename length limit via `kMaxFilenameChars`
  - Checks whether to "createNewFile" by comparing recent filename against default; reverts to new-file mode if default

## Control Flow Notes
Entry point is `save_game_full_auto()`, typically called from frame updates or explicit autosave triggers. Filename generation is lazy (only computed if creating new file). File existence checks are per-variant, polling the filesystem until a unique name is found. Actual serialization is delegated to `save_game_file()` (defined elsewhere).

## External Dependencies
- **FileHandler.h**: `FileSpecifier`, `DirectorySpecifier` classes
- **game_wad.h**: `save_game_file()`, `get_current_saved_game_name()`
- **TextStrings.h**: `TS_GetCString()` (string resource lookup)
- **map.h**: `static_world->level_name` (current level name global)
- **shell.h**: `screen_printf()` (on-screen message display)
- **ctype.h**: `isalnum()` (character classification)
- Standard C/C++: `memset()`, `sprintf()`, `strlen()`, `strcmp()`, `strcpy()`
