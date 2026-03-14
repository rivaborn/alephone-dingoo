# Source_Files/Misc/interface.cpp

## File Purpose

Central game state machine and shell controller for Aleph One. Manages the entire game lifecycle from startup (intro screens) through main menu, game initialization, level transitions, and shutdown. Coordinates between game systems (network, graphics, audio, players) and maintains UI state.

## Core Responsibilities

- **Game state transitions**: Validates and manages state changes (intro ΓåÆ menu ΓåÆ game in progress ΓåÆ epilogue ΓåÆ quit)
- **Screen display pipeline**: Manages cinematic fades, resource loading, and screen transitions
- **Game initialization**: Constructs player start data, handles single/multiplayer setup, resumes saved games
- **Menu/interface input**: Processes button clicks, manages main menu display
- **Network game coordination**: Gathers players, joins sessions, handles resume-game logic
- **Preference application**: Reflects graphics settings changes without restarting
- **Background task suppression**: Controls when background tasks (Lua, music) are allowed during transitions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `game_state` | struct | Central state machine: current state, phase counter, flags, player controller type, netgame mic permission |
| `screen_data` | struct | Maps screen resource base ID, count, and duration for display sequences |
| `player_start_data` | struct (imported from map.h) | Player identity, team, color, name for game initialization |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `game_state` | `struct game_state` | static | Main state machine instance |
| `interface_fade_in_progress` | bool | static | Tracks active cinematic fade animation |
| `interface_fade_type` | short | static | Type of fade (fade in/out) |
| `current_picture_clut` | `color_table*` | static | CLUT for currently displayed screen picture |
| `current_picture_clut_depth` | short | static | Bit depth of current picture CLUT (8 or 16) |
| `animated_color_table` | `color_table*` | static | Working CLUT during fade animation |
| `DraggedReplayFile` | `FileSpecifier` | static | Dragged replay file pending playback |
| `display_screens[]` | `screen_data[]` | static const | Screen sequence definitions (intro, menu, prologue, epilogue, credits, etc.) |

## Key Functions / Methods

### initialize_game_state
- **Purpose**: Entry point; initialize game state machine to intro screens, reset flags, display introduction.
- **Signature**: `void initialize_game_state(void)`
- **Inputs**: None
- **Outputs/Return**: None (void)
- **Side effects**: Sets `game_state` to intro, calls `display_introduction()`, checks insecure Lua and alerts if enabled
- **Calls**: `display_introduction()`, `toggle_menus()`, `alert_user()`
- **Notes**: Called once at startup; suppresses background tasks initially

### idle_game_state
- **Purpose**: Main game loop update function; advance screen timers, process state transitions, update fade animations
- **Signature**: `bool idle_game_state(uint32 ticks)`
- **Inputs**: `ticks` ΓÇö elapsed real-time ticks since last call
- **Outputs/Return**: bool ΓÇö true if screen/state changed
- **Side effects**: Updates `game_state.phase`, calls state handlers, may call `next_game_screen()`, `update_interface_fades()`, or game-specific update logic
- **Calls**: `next_game_screen()`, `update_interface_fades()`, screen-specific handlers
- **Notes**: Throttles to 30 ticks per second; handles countdown-style phase timing

### set_game_state
- **Purpose**: Validated state transition; queues state changes that may require deferred processing at idle time
- **Signature**: `void set_game_state(short new_state)`
- **Inputs**: `new_state` ΓÇö new game state constant (e.g., `_game_in_progress`, `_close_game`)
- **Outputs/Return**: None (void)
- **Side effects**: Updates `game_state.state` and `game_state.phase`; may call `finish_game()` immediately for some transitions
- **Calls**: `finish_game()`, possibly deferred processing functions
- **Notes**: Special handling for state transitions from `_game_in_progress` (e.g., defers demo switches and reverts to idle time); other states transition immediately

### portable_process_screen_click
- **Purpose**: Handle mouse clicks on main menu buttons; draw pressed state and execute menu commands
- **Signature**: `void portable_process_screen_click(short x, short y, bool cheatkeys_down)`
- **Inputs**: `x, y` ΓÇö click coordinates; `cheatkeys_down` ΓÇö modifier key state
- **Outputs/Return**: None (void)
- **Side effects**: May stop fades, draw button states, execute menu commands (start game, load, preferences, etc.)
- **Calls**: `point_in_rectangle()`, `get_interface_rectangle()`, `enabled_item()`, `draw_button()`, `do_menu_item_command()`
- **Notes**: Includes SDL/Dingoo-specific screen offset handling; modal mouse-tracking loop

### construct_single_player_start / construct_multiplayer_starts
- **Purpose**: Build player start data from preferences (single player) or network game info (multiplayer)
- **Signature**: `static void construct_single_player_start(player_start_data* outStartArray, short* outStartCount)` (and multiplayer variant)
- **Inputs**: Output array pointers
- **Outputs/Return**: Populates `outStartArray[0]` and sets `*outStartCount`
- **Side effects**: Copies player name from preferences, reads network player data
- **Calls**: `NetGetNumberOfPlayers()`, `NetGetPlayerData()`, `NetGetPlayerIdentifier()`
- **Notes**: Part of generalized game startup system; handles name encoding and behavior modifier flags

### match_starts_with_existing_players
- **Purpose**: Reconcile player start data with existing player objects (for resume-game); mark unmatched players as zombies
- **Signature**: `void match_starts_with_existing_players(player_start_data* ioStartArray, short* ioStartCount)`
- **Inputs**: Start array and count (modified in-place)
- **Outputs/Return**: None (void); reorders start array to match existing players
- **Side effects**: Matches starts to players by name; creates new start entries for unmatched players; reorders to align indices
- **Calls**: `get_player_data()`, `strcmp()`, `memcpy()`, `strcpy()`
- **Notes**: Greedy matching: first by name, then arbitrarily; handles both resume (existing players) and new game (empty roster)

### synchronize_players_with_starts
- **Purpose**: Apply start data to existing players and create new players; mark players without starts as zombies
- **Signature**: `static void synchronize_players_with_starts(const player_start_data* inStartArray, short inStartCount)`
- **Inputs**: Start array and count
- **Outputs/Return**: None (void); modifies global player array
- **Side effects**: Updates player appearance (team, color, name, identifiers), sets zombie status, calls `new_player()` for missing players
- **Calls**: `get_player_data()`, `new_player()`, `player_identifier_value()`, behavior flag setter macros
- **Notes**: Idempotent regarding player count if starts count matches; used in both new-game and resume-game flows

### make_restored_game_relevant
- **Purpose**: Finalize a loaded/resumed game (single or multiplayer); set random seed, sync players, initialize game info
- **Signature**: `static bool make_restored_game_relevant(bool inNetgame, const player_start_data* inStartArray, short inStartCount)`
- **Inputs**: Whether networked; start array and count
- **Outputs/Return**: bool ΓÇö true if successful, false if map entry fails
- **Side effects**: Sets `game_is_networked`, calls `set_random_seed()`, `synchronize_players_with_starts()`, `entering_map()`, `reset_motion_sensor()`. For netgames, installs microphone, syncs difficulty/game info from network, standardizes behavior modifiers. For single-player, restores custom behavior.
- **Calls**: `entering_map()`, `synchronize_players_with_starts()`, `install_network_microphone()`, `reset_motion_sensor()`, `clean_up_after_failed_game()`
- **Notes**: Critical path for resume-game feature; must set random seed before player creation

### join_networked_resume_game
- **Purpose**: Join a network game that is resuming (not starting fresh); receive game/map data from gatherer and initialize
- **Signature**: `static bool join_networked_resume_game()`
- **Inputs**: None (reads from network API and file system)
- **Outputs/Return**: bool ΓÇö true if successful
- **Side effects**: Receives and loads game WAD from network, loads scenario, receives map and player starts, applies `make_restored_game_relevant()`, cleans up on failure
- **Calls**: `NetReceiveGameData()`, `load_game_from_file()`, `choose_saved_game_to_load()`, `make_restored_game_relevant()`, `clean_up_after_failed_game()`
- **Notes**: Mirrors single-player load flow but uses network-received data; deferred script execution

### begin_game / finish_game
- **Purpose**: `begin_game()` starts a new game after menu; `finish_game()` ends game and returns to menu/quit
- **Signature**: `static bool begin_game(short user, bool cheat)` / `static void finish_game(bool return_to_main_menu)`
- **Inputs**: `user` ΓÇö controller type (_single_player, _network_player); `cheat` ΓÇö cheat mode flag / `return_to_main_menu` ΓÇö whether to show menu or quit screens
- **Outputs/Return**: bool (begin_game) ΓÇö true if successful
- **Side effects**: Resets screen, initializes player/world, may show chapter screen, constructs player starts, calls entering_map, updates game state. For finish, stops recording/sounds, displays menu or quit screens
- **Calls**: `reset_screen()`, `initialize_map_for_new_game()`, `construct_*_player_start()`, `entering_map()`, `try_and_display_chapter_screen()`, `new_game()` (finish_game calls this indirectly)
- **Notes**: `begin_game` is called after networking or load dialogs complete

### next_game_screen
- **Purpose**: Advance to next screen in current sequence (intro, credits, etc.) or trigger state transitions when sequence ends
- **Signature**: `static void next_game_screen(void)`
- **Inputs**: None (reads `game_state`)
- **Outputs/Return**: None (void)
- **Side effects**: Increments `game_state.current_screen`; if at end of sequence, transitions to next state (e.g., fade out quit screens, show main menu). Restarts intro music on specified screen. Checks image existence and sets phase to 0 if missing.
- **Calls**: `get_screen_data()`, `stop_interface_fade()`, `Music::instance()->RestartIntroMusic()`, `images_picture_exists()`, `display_screen()`, `interface_fade_out()`
- **Notes**: Driven by `idle_game_state()` countdown; skips missing picture resources gracefully

### display_screen
- **Purpose**: Load and display a picture resource with cinematic fade-in
- **Signature**: `static void display_screen(short base_pict_id)`
- **Inputs**: `base_pict_id` ΓÇö base resource ID (actual ID = base + current_screen offset)
- **Outputs/Return**: None (void)
- **Side effects**: Calculates CLUT, fades out old picture, draws new picture, fades in new CLUT, updates `current_picture_clut` and depth, calls `start_interface_fade()`
- **Calls**: `images_picture_exists()`, `stop_interface_fade()`, `interface_fade_out()`, `calculate_picture_clut()`, `full_fade()`, `draw_full_screen_pict_resource_from_images()`, `start_interface_fade()`
- **Notes**: Handles bit-depth mismatch by recalculating CLUT; if picture missing, calls `next_game_screen()` to skip

### interface_fade_out / start_interface_fade / update_interface_fades
- **Purpose**: Manage cinematic color-table fade animations (fade in/out) with optional music fade
- **Signature**: `void interface_fade_out(short pict_resource_number, bool fade_music)` / `static void start_interface_fade(short type, struct color_table *original_color_table)` / `static void update_interface_fades(void)`
- **Inputs**: Resource ID and music flag / fade type and original CLUT / none
- **Outputs/Return**: None (void)
- **Side effects**: Allocates/frees animated color tables, calls fade engine functions, pauses/resumes music, hides/shows cursor, paints window black
- **Calls**: `delete/new color_table`, `explicit_start_fade()`, `update_fades()`, `Music::instance()->FadeOut/Playing/Pause/Idle()`, `paint_window_black()`, `hide_cursor()`, `show_cursor()`
- **Notes**: State tracked in `interface_fade_in_progress`; update is called every frame; fade-out happens synchronously (blocks on update_fades loop)

### handle_network_game
- **Purpose**: Orchestrate network gather or join flow, then call appropriate game start or resume function
- **Signature**: `static void handle_network_game(bool gatherer)`
- **Inputs**: `gatherer` ΓÇö true to gather players, false to join
- **Outputs/Return**: None (void)
- **Side effects**: Sets state to `_displaying_network_game_dialogs`, calls `network_gather()` or `network_join()`, calls `NetStart()`, calls `begin_game()` or `join_networked_resume_game()`, restores colors on cancel
- **Calls**: `force_system_colors()`, `network_gather()`, `network_join()`, `NetStart()`, `begin_game()`, `join_networked_resume_game()`, `clean_up_after_failed_game()`, `display_main_menu()`
- **Notes**: Result of `network_join()` distinguishes new-game vs resume-game join

### do_preferences
- **Purpose**: Apply user preference changes; re-initialize graphics if bit depth changed
- **Signature**: `void do_preferences(void)`
- **Inputs**: None
- **Outputs/Return**: None (void)
- **Side effects**: Calls graphics preference handler; if bit depth changed, reinitializes screen and redisplays menu; if other screen mode changed, updates screen mode
- **Calls**: `handle_preferences()`, `Screen::instance()->Initialize()`, `change_screen_mode()`, `paint_window_black()`, `display_main_menu()`
- **Notes**: Called from menu; minimal impact if only non-critical settings changed

### should_restore_game_networked
- **Purpose**: Modal dialog asking user whether to resume single-player or multiplayer
- **Signature**: `size_t should_restore_game_networked()`
- **Inputs**: None (reads dynamic_world->player_count to pre-select)
- **Outputs/Return**: size_t ΓÇö UNONE (cancel), 0 (single-player), 1 (multiplayer)
- **Side effects**: Shows dialog with toggle and buttons
- **Calls**: Dialog widget creation and execution
- **Notes**: Cursor is hidden when called; called after game is loaded from file but before final initialization

## Control Flow Notes

**Initialization**: `initialize_game_state()` ΓåÆ state = `_display_intro_screens` ΓåÆ `display_introduction()` called

**Main Loop**: `idle_game_state()` is polled each frame; advances screen counters and triggers state transitions:
- **Intro Phase**: Display intro screens on timer ΓåÆ transition to main menu
- **Menu Phase**: Main menu displayed; user clicks button ΓåÆ `portable_process_screen_click()` ΓåÆ `do_menu_item_command()` ΓåÆ state transitions (new game, load, network, prefs, quit)
- **Game Startup**: Gather/join network (if applicable) ΓåÆ `begin_game()` or `join_networked_resume_game()` ΓåÆ state = `_game_in_progress` (handed off to game loop)
- **Game In Progress**: State machine defers changes to idle; `set_game_state()` queues transitions
- **Game End**: `set_game_state(_close_game/_quit_game)` ΓåÆ `finish_game()` ΓåÆ display epilogue/credits/quit screens or main menu
- **Epilogue/Credits**: Timed screens, then return to menu or quit

**Fade System**: Display/menu transitions use `interface_fade_out()` for music fade and color fade to black, then `start_interface_fade()` for fade-in of next screen. Fade updates happen in `update_interface_fades()` called from idle.

**Network Resume**: Special path via `join_networked_resume_game()` ΓåÆ `make_restored_game_relevant()` ΓåÆ calls same map-entry logic as single-player resume but with network-provided data.

## External Dependencies

- **map.h**: Player start data, game world structures, level loading (`entering_map`, `initialize_map_for_new_game`, `load_level_from_map`)
- **player.h**: Player management (`new_player`, `get_player_data`, `set_local_player_index`)
- **network.h**: Network game coordination (`network_gather`, `network_join`, `NetStart`, `NetReceiveGameData`, microphone functions)
- **screen_drawing.h**: Rectangle/button drawing, interface colors
- **Music.h**: Music playback control (`Music::instance()->RestartIntroMusic()`, `FadeOut`, `Playing`, `Pause`, `Idle`)
- **fades.h**: Color-table fade engine (`explicit_start_fade`, `update_fades`, `full_fade`)
- **game_window.h**: Screen/window management
- **Mixer.h**: Sound playback
- **images.h**: Picture resource existence check and loading
- **sdl_dialogs.h / sdl_widgets.h**: Dialog and widget system for preferences, network dialogs
- **interface_menus.h**: Menu command dispatch (`do_menu_item_command`)
- **XML_LevelScript.h**: Lua scripting and end-game script execution
- **lua_hud_script.h**: HUD script support
- **preferences.h**: User preferences (color, difficulty, controller setup)
- **FileHandler.h**: File I/O abstraction
- **network_sound.h**: Network microphone/speaker (`install_network_microphone`, `remove_network_microphone`)
- **motion_sensor.h**: `reset_motion_sensor()` for camera reset on level entry
- **vbl.h**: Vertical-blank and heartbeat timing
- **game_errors.h**: Error handling and reporting

**Defined Elsewhere** (external symbols used):
- `dynamic_world`, `static_world` ΓÇö game world data (map.h)
- `local_player`, `current_player` ΓÇö active player references (player.h)
- `get_shape_surface()`, `load_collections()` ΓÇö shape/texture loading (shell.h)
- `load_game_from_file()`, `choose_saved_game_to_load()` ΓÇö game loading (preprocess_map_mac.c)
- `new_game()`, `entering_map()`, `load_level_from_map()` ΓÇö world initialization (game_wad.c, map.c)
- `update_screen_window()`, `update_interface_display()` ΓÇö rendering (game_window.c, screen_drawing.c)
- `Music::instance()`, `Mixer::instance()` ΓÇö singleton audio managers
- `Screen::instance()->Initialize()` ΓÇö screen mode initialization
- Network API (`NetGetNumberOfPlayers`, `NetGetPlayerData`, etc.) ΓÇö network subsystem (network.c)
