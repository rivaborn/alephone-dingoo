# Source_Files/Files/preprocess_map_mac.cpp

## File Purpose
Mac-specific map and game file preprocessing routines. Handles locating default resource files (maps, physics, shapes, sounds) via recursive directory search, managing save game dialogs, and creating/embedding thumbnail previews in saved game files.

## Core Responsibilities
- Recursively search application directory for required game resource files by type code
- Present file dialogs for load/save game operations
- Generate and embed overhead map thumbnails in saved game files
- Add metadata resources (application name, etc.) to save files
- Initialize game state from selected save files
- Pause/resume game during file I/O operations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `file_type_to_find_rec` | struct | Tracks search state for a single file type (found flag, file result, output location) |
| `FileFinder` | class (external) | Abstraction for recursive file system traversal with callbacks |
| `overhead_map_data` | struct (external) | Configuration for rendering overhead map preview (scale, origin, dimensions) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `file_typecodes_to_find[]` | int[] | static | Array of file type codes to search for (scenario, shapes, sounds, physics) |
| `file_types_to_find[]` | file_type_to_find_rec[] | static | Parallel array tracking results for each file type during search |
| `world_pixels` | GWorldPtr | extern | Graphics offscreen buffer used to render thumbnails |

## Key Functions / Methods

### get_default_file_specs
- **Signature:** `void get_default_file_specs(FileSpecifier* outMapSpec, FileSpecifier* outShapesSpec, FileSpecifier* outSoundsSpec, FileSpecifier* outPhysicsSpec)`
- **Purpose:** Unified search for all default resource files; replaces individual get_default_*_spec functions.
- **Inputs:** Four output pointers (any can be NULL to skip search for that type).
- **Outputs/Return:** Populates output FileSpecifier objects; fatal error if required files (map/shapes/sounds) not found.
- **Side effects:** Recursively searches app directory tree; shows fatal alert if critical files missing.
- **Calls:** `search_from_directory()`, `found_some_file_callback()`, `Files_GetRootDirectory()`, `alert_user()`.
- **Notes:** Searches from app root, stops when all required files found or directory fully traversed. Physics file is optional.

### search_from_directory
- **Signature:** `static void search_from_directory(DirectorySpecifier& BaseDir)`
- **Purpose:** Performs recursive file search from a base directory using FileFinder callback.
- **Inputs:** Base directory to start search.
- **Outputs/Return:** None (updates static `file_types_to_find[]` via callback).
- **Side effects:** Populates file_types_to_find[] array; may show assertion if FileFinder fails.
- **Calls:** `FileFinder::Find()`, callback `found_some_file_callback()`.
- **Notes:** Hardcoded to always recurse; callback-driven pattern allows early termination.

### save_game
- **Signature:** `bool save_game(void)`
- **Purpose:** Initiates save-game workflow: pause, show dialog, save file, resume.
- **Inputs:** None.
- **Outputs/Return:** true if save succeeded.
- **Side effects:** Pauses game, shows cursor, calls `save_game_file()`, calls `add_finishing_touches_to_save_file()`, resumes game.
- **Calls:** `pause_game()`, `show_cursor()`, `get_current_saved_game_name()`, `SaveFile.WriteDialogAsync()`, `save_game_file()`, `hide_cursor()`, `resume_game()`.
- **Notes:** Uses async dialog to allow background sound playback.

### add_overhead_thumbnail
- **Signature:** `static void add_overhead_thumbnail(FileSpecifier &File)`
- **Purpose:** Renders overhead map view as Pict resource and embeds in save file for Nav Services preview.
- **Inputs:** Target save file specifier.
- **Outputs/Return:** None (modifies file via resource fork).
- **Side effects:** Uses `world_pixels` GWorld for offscreen rendering; modifies file resource fork.
- **Calls:** `GetGWorld()`, `SetGWorld()`, `OpenPicture()`, `_render_overhead_map()`, `NavCreatePreview()`, `CloseResFile()`, `KillPicture()`.
- **Notes:** Creates two Pict resources (thumbnail and preview); requires NavServices 2.0+; skips if unavailable.

## Control Flow Notes
- **Startup:** `get_default_file_specs()` called early to locate resource WAD files.
- **Save:** `save_game()` ΓåÆ `SaveFile.WriteDialogAsync()` ΓåÆ `save_game_file()` ΓåÆ `add_finishing_touches_to_save_file()` ΓåÆ `add_overhead_thumbnail()`.
- **Load:** `choose_saved_game_to_load()` ΓåÆ file dialog returns selected save.
- **Resource embedding:** Thumbnail added to save file's resource fork after main game data written.

## External Dependencies
- **FileHandler.h** ΓÇô FileSpecifier and DirectorySpecifier classes; file dialog abstraction.
- **world.h** ΓÇô world_point2d, world_point3d, coordinate math.
- **overhead_map.h** ΓÇô overhead_map_data, _render_overhead_map().
- **game_wad.h** ΓÇô save_game_file(), get_current_saved_game_name().
- **Mac APIs** (via macintosh_cseries.h) ΓÇô FSSpec, GWorld, QuickDraw/Resource Manager (FSpOpenResFile, AddResource, etc.), Navigation Services.
- **shell.h** ΓÇô getcstr(), pause_game(), resume_game(), show/hide_cursor().
