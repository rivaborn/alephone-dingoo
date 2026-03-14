# Source_Files/Files/game_wad.cpp

## File Purpose
Handles loading and saving complete game maps and game state from/to WAD (Where's All the Data) files. Manages map file selection, level initialization, game state serialization, and network map distribution for the Aleph One engine.

## Core Responsibilities
- Select and open map files; manage current map file state
- Load level geometry and objects from WAD chunks
- Initialize new games with player placement and game parameters
- Build and serialize game state to save-game WAD files
- Export maps as playable WAD files
- Handle network map data transfer between players
- Restore previously saved games
- Query map metadata (entry points, checksums, level names)
- Manage physics model and Lua script loading

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `revert_game_info` | struct | Holds game state for restoration (game data, players, entry point, saved game file) |
| `save_game_data` | struct | Metadata for a saveable/loadable data chunk (tag, unit size, loaded-by-level flag) |
| `export_data` | array | List of saveable chunks for map export |
| `save_data` | array | List of saveable chunks for game save files |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MapFileSpec` | FileSpecifier | file-static | Path to currently loaded map file |
| `file_is_set` | bool | file-static | Whether a valid map file is selected |
| `PhysicsModelLoadedEarlier` | bool | file-static | Tracks if physics model was loaded from previous level |
| `revert_game_data` | revert_game_info | file-static | Saved game state for reverting to after level changes |
| `static_platforms` | vector<static_platform_data> | file-static | Cached platform definitions for level export |

## Key Functions / Methods

### set_map_file
- Signature: `void set_map_file(FileSpecifier& File)`
- Purpose: Select the map file to use for level loading; initialize level scripts and scenario images
- Inputs: File specifier for the map
- Outputs: None (modifies global state)
- Side effects: Loads level scripts, sets scenario images, triggers parameter restoration
- Calls: `RunRestorationScript()`, `set_scenario_images_file()`, `LoadLevelScripts()`, `clear_game_error()`
- Notes: Must be called before attempting to load levels

### load_level_from_map
- Signature: `bool load_level_from_map(short level_index)`
- Purpose: Load a single level from the current map file (or restore saved game if `level_index==NONE`)
- Inputs: Level index (0-based) or `NONE` to restore save game
- Outputs: True if successful
- Side effects: Opens map file, reads header, extracts indexed WAD, processes all map data, frees WAD
- Calls: `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `process_map_wad()`, `close_wad_file()`
- Notes: Error state set on failure; restoring game uses index 0 of WAD

### new_game
- Signature: `bool new_game(short number_of_players, bool network, game_data *game_information, player_start_data *player_start_information, entry_point *entry_point)`
- Purpose: Initialize a new game session with players and load the starting level
- Inputs: Player count, networking flag, game setup data, player start data, entry point
- Outputs: True if successful
- Side effects: Initializes map, sets random seed, creates players, enters level, resets action queues, initializes motion sensor and chase cam
- Calls: `initialize_map_for_new_game()`, `set_random_seed()`, `goto_level()`, `new_player()`, `entering_map()`, `reset_motion_sensor()`, `ChaseCam_Initialize()`
- Notes: Invokes Lua ResetPassedLua if available; saves revert game info for later restoration

### build_save_game_wad
- Signature: `static wad_data *build_save_game_wad(wad_header *header, int32 *length)`
- Purpose: Pack all current game state (geometry, objects, monsters, players, dynamics) into a WAD for saving
- Inputs: WAD header template, output length pointer
- Outputs: Pointer to allocated WAD (caller must free), sets length
- Side effects: Allocates memory for packed data; recalculates map counts
- Calls: `create_empty_wad()`, `recalculate_map_counts()`, `tag_to_global_array_and_size()`, `append_data_to_wad()`, `calculate_wad_length()`
- Notes: Handles ~30 different data types; each is converted to packed binary format

### build_export_wad
- Signature: `static wad_data *build_export_wad(wad_header *header, int32 *length)`
- Purpose: Pack map geometry and static objects for export; resets platforms/lines to initial states
- Inputs: WAD header template, output length pointer
- Outputs: Pointer to allocated WAD, sets length
- Side effects: Saves and restores platform/polygon/line lists (for export coherence)
- Calls: `create_empty_wad()`, `recalculate_map_counts()`, `export_tag_to_global_array_and_size()`, `append_data_to_wad()`, `calculate_wad_length()`
- Notes: Clears platform states and variable-elevation lines before export; restores afterward

### complete_loading_level
- Signature: `void complete_loading_level(short *_map_indexes, size_t map_index_count, uint8 *_platform_data, size_t platform_data_count, uint8 *actual_platform_data, size_t actual_platform_data_count, short version)`
- Purpose: Finalize level setup after geometry is loaded: load redundant indices, add platforms and scenery, initialize lighting
- Inputs: Map index data, platform data (two formats), data version
- Outputs: None (modifies global structures)
- Side effects: Populates map indices, platforms, scenery; recalculates redundant polygon data
- Calls: `load_redundant_map_data()`, `scan_and_add_platforms()`, `scan_and_add_scenery()`, `guess_side_lightsource_indexes()`, `recalculate_redundant_map()`
- Notes: Called after all geometry is loaded; must occur before entering map

### get_indexed_entry_point / get_entry_points
- Signature: `bool get_indexed_entry_point(entry_point *ep, short *index, int32 type)` / `bool get_entry_points(vector<entry_point> &vec, int32 type)`
- Purpose: Query map entry points matching given game type flags (single-player, multiplayer, etc.)
- Inputs: Entry point struct (or vector), index/type flags; type flags indicate valid game modes
- Outputs: True if found; populates entry point data and advances index
- Side effects: Opens map file, reads header, extracts directory or legacy map info data
- Calls: `open_wad_file_for_reading()`, `read_wad_header()`, `read_directory_data()`, `unpack_directory_data()`, `extract_type_from_wad()`, `unpack_static_data()`, `close_wad_file()`
- Notes: Supports both new (directory) and old (legacy) WAD formats

## Control Flow Notes
**Initialization sequence:**
1. `set_map_file()` ΓÇö select map file at startup or when user chooses
2. `load_level_from_map(entry_point.level_number)` ΓÇö load map geometry/objects from WAD
3. `process_map_wad()` [not in this file, called by load_level_from_map] ΓÇö deserialize all chunks
4. `complete_loading_level()` ΓÇö finalize platforms, scenery, lighting
5. `new_game()` ΓåÆ `goto_level()` ΓåÆ `entering_map()` ΓÇö place players and start gameplay

**Save/restore:**
- `new_game()` calls `setup_revert_game_info()` to save state for later `revert_game()`
- `build_save_game_wad()` serializes current world state; written by caller
- `load_level_from_map(NONE)` restores from save WAD

**Network:**
- `process_net_map_data()` ΓÇö receives inflated map WAD from network; unpacks and processes
- `get_map_for_net_transfer()` ΓÇö reads map file and provides flat data buffer for sending to peers

## External Dependencies
- **map.h** ΓÇö defines map structures (polygon, line, endpoint, etc.)
- **monsters.h, projectiles.h, effects.h** ΓÇö entity type definitions and packing/unpacking
- **player.h** ΓÇö player data and physics model info
- **network.h** ΓÇö networking function declarations
- **platforms.h** ΓÇö platform/door types
- **wad.h, FileHandler.h** ΓÇö WAD file I/O
- **Packing.h** ΓÇö serialization (pack/unpack) routines for game data
- **shell.h, preferences.h, ChaseCam.h, render.h** ΓÇö engine subsystems
- **XML_LevelScript.h, Music.h, SoundManager.h** ΓÇö script and audio management
- **computer_interface.h** ΓÇö terminal state management
- **game_errors.h, game_window.h, interface.h** ΓÇö UI and error reporting
