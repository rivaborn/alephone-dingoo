# Source_Files/Files/preprocess_map_sdl.cpp

## File Purpose
Save game and asset file management for SDL-based game engine. Provides utilities to locate default game assets (maps, physics, shapes, sounds, music, themes) and handle save/load game workflows via file dialogs.

## Core Responsibilities
- Locate default game asset files by searching data paths
- Distinguish between critical assets (map, shapes) and optional assets (physics, sounds, music)
- Display file dialogs for save game selection and confirmation
- Coordinate pause/resume and cursor visibility during save operations
- Provide extensibility hooks for platform-specific save file post-processing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FileSpecifier | class (imported) | Encapsulates file path specifications; used throughout for all file operations |
| DirectorySpecifier | class (imported) | Encapsulates directory path; used in data_search_path vector |
| string | typedef (STL) | Used for filename strings in asset lookup |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| data_search_path | vector<DirectorySpecifier> | extern (from shell_sdl.cpp) | Search path(s) for locating default data files |

## Key Functions / Methods

### get_default_spec (overload 1)
- **Signature:** `static bool get_default_spec(FileSpecifier &file, const string &name)`
- **Purpose:** Utility to search data_search_path for a file by name
- **Inputs:** Reference to FileSpecifier (output); filename as string
- **Outputs/Return:** Boolean (true if found)
- **Side effects:** Modifies `file` parameter to point to found file; I/O via FileSpecifier::Exists()
- **Calls:** FileSpecifier::Exists()
- **Notes:** Iterates until first match or end of path; no error on failure (caller decides criticality)

### get_default_spec (overload 2)
- **Signature:** `static bool get_default_spec(FileSpecifier &file, int type)`
- **Purpose:** Type-code wrapper; converts enum to filename string, delegates to overload 1
- **Inputs:** FileSpecifier output ref; integer type code (from strFILENAMES enum)
- **Outputs/Return:** Boolean
- **Calls:** getcstr() [string resource lookup]; overload 1
- **Notes:** Decouples callers from filename string constants

### get_default_map_spec
- **Signature:** `void get_default_map_spec(FileSpecifier &file)`
- **Purpose:** Locate the default map file (critical asset)
- **Inputs:** FileSpecifier output reference
- **Outputs/Return:** None (sets file by reference)
- **Side effects:** Calls alert_user(fatalError, ...) if file not found; halts on missing map
- **Calls:** get_default_spec(file, filenameDEFAULT_MAP)
- **Notes:** Fatal failure if map not foundΓÇögame cannot proceed

### get_default_physics_spec
- **Signature:** `void get_default_physics_spec(FileSpecifier &file)`
- **Purpose:** Locate physics model (optional asset)
- **Side effects:** Silent failure; no alert if missing
- **Calls:** get_default_spec(file, filenamePHYSICS_MODEL)

### get_default_shapes_spec
- **Signature:** `void get_default_shapes_spec(FileSpecifier &file)`
- **Purpose:** Locate shapes file (critical asset)
- **Side effects:** Fatal alert if missing
- **Calls:** get_default_spec(file, filenameSHAPES8)

### get_default_sounds_spec
- **Signature:** `void get_default_sounds_spec(FileSpecifier &file)`
- **Purpose:** Locate sounds file (optional)
- **Calls:** get_default_spec(file, filenameSOUNDS8)

### get_default_music_spec
- **Signature:** `bool get_default_music_spec(FileSpecifier &file)`
- **Purpose:** Locate music file
- **Outputs/Return:** Boolean success
- **Calls:** get_default_spec(file, filenameMUSIC)

### get_default_theme_spec
- **Signature:** `bool get_default_theme_spec(FileSpecifier &file)`
- **Purpose:** Locate theme file in Themes subdirectory
- **Outputs/Return:** Boolean success
- **Side effects:** Constructs path as "Themes" + theme filename string
- **Calls:** getcstr() [for theme name]; get_default_spec()
- **Notes:** Path construction pattern differs from simpler filename lookups

### choose_saved_game_to_load
- **Signature:** `bool choose_saved_game_to_load(FileSpecifier &saved_game)`
- **Purpose:** Display file browser for user to select a save game to load
- **Inputs:** FileSpecifier output reference
- **Outputs/Return:** Boolean (true if user selected, false if cancelled)
- **Side effects:** UI I/O via FileSpecifier::ReadDialog()
- **Calls:** saved_game.ReadDialog(_typecode_savegame)

### save_game
- **Signature:** `bool save_game(void)`
- **Purpose:** Main save game workflow: pause, prompt user, save data, resume
- **Outputs/Return:** Boolean success
- **Side effects:** Pauses game; shows/hides cursor; I/O via async file dialog
- **Calls:** 
  - pause_game(), resume_game()
  - show_cursor(), hide_cursor()
  - get_current_saved_game_name()
  - file.GetName(), file.WriteDialogAsync(), save_game_file()
- **Notes:** Async write dialog allows sound to continue during save; user can provide custom filename

### add_finishing_touches_to_save_file
- **Signature:** `void add_finishing_touches_to_save_file(FileSpecifier &file)`
- **Purpose:** Platform-specific post-processing of save file (stub)
- **Notes:** Mac version stores overhead thumbnail and level name in resource fork; not implemented for SDL version

## Control Flow Notes
**Initialization/Loading:** Asset location functions (get_default_*_spec) called at engine startup to load critical resources (map, shapes) and optional resources (physics, sounds, music).

**Save Flow:** save_game() coordinates a pauseΓåÆUIΓåÆwriteΓåÆresume sequence, using async dialog to maintain audio during save. User can override default save filename.

**Load Flow:** choose_saved_game_to_load() is called to let user browse and select a saved game file before deserialization.

## External Dependencies
- **cseries.h** ΓÇô Core engine infrastructure (macros, constants, string utilities)
- **FileHandler.h** ΓÇô FileSpecifier, DirectorySpecifier classes for cross-platform file I/O
- **world.h, map.h** ΓÇô Game world and map data structures
- **shell.h, interface.h** ΓÇô Game state and UI interface (pause_game, show_cursor, etc.)
- **game_wad.h** ΓÇô Game WAD serialization (save_game_file, get_current_saved_game_name)
- **game_errors.h** ΓÇô Error codes and alert macros (alert_user, fatalError, badExtraFileLocations)
- **vector** (STL) ΓÇô Used in data_search_path
- **data_search_path** (extern from shell_sdl.cpp) ΓÇô Multi-path search list for asset discovery
