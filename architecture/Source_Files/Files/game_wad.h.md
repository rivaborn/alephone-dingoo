# Source_Files/Files/game_wad.h

## File Purpose
Header declaring public interfaces for game WAD file operations in the Aleph One engine. Manages save/load functionality, level/map loading, and associated file specifications.

## Core Responsibilities
- Save and export game state to WAD files
- Load and process WAD files (maps, physics, scripting)
- Manage current map file context and file specifications
- Pause/resume game execution
- Query embedded level metadata (physics, Lua scripts)
- Validate map file integrity via checksums

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class (forward decl.) | Abstracts file path/reference specifications across platforms |
| `wad_data` | struct (forward decl.) | Container for WAD file data (maps, resources, metadata) |

## Global / File-Static State
None.

## Key Functions / Methods

### save_game_file
- Signature: `bool save_game_file(FileSpecifier& File);`
- Purpose: Persist current game state to disk
- Inputs: File specification (output location)
- Outputs/Return: Success/failure boolean
- Side effects: I/O (writes file); modifies save game state

### process_map_wad
- Signature: `bool process_map_wad(struct wad_data *wad, bool restoring_game, short version);`
- Purpose: Parse and apply WAD file contents to load a map/level
- Inputs: WAD data pointer, restoration flag (true if loading saved game), version number
- Outputs/Return: Success/failure boolean
- Side effects: Loads level data, physics, scripts; modifies game state
- Notes: Used for both initial level load and netgame resume

### get_map_file
- Signature: `FileSpecifier& get_map_file(void);`
- Purpose: Retrieve current active map file specification
- Outputs/Return: Reference to active map file
- Notes: Returns persistent reference (not a copy)

### level_has_embedded_physics_lua
- Signature: `void level_has_embedded_physics_lua(int Level, bool& HasPhysics, bool& HasLua);`
- Purpose: Query whether a level contains embedded physics/Lua scripts
- Inputs: Level index
- Outputs/Return: Two output booleans (physics present, Lua present)

### Remaining functions
Trivial or self-documenting: `pause_game()`, `resume_game()`, `export_level()`, `set_map_file()`, `get_current_saved_game_name()`, `set_saved_game_name_to_default()`, `match_checksum_with_map()`, `get_savegame_filedesc()`, `add_finishing_touches_to_save_file()`.

## Control Flow Notes
Likely invoked during game initialization (load level), runtime (save/pause), and shutdown. `process_map_wad()` is a core entry point for level loading; checksum validation suggests integrity checks before level load.

## External Dependencies
- `FileSpecifier` class (OOP file handler; defined elsewhere)
- `wad_data` struct (WAD container; defined elsewhere)
- References to Chris Pruett's Pfhortran (scripting system)
