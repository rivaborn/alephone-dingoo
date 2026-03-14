# Source_Files/Misc/interface.h

## File Purpose
Master interface header for the Aleph One game engine. Declares game state management, menu/UI systems, shape and texture collection loading, input handling, recording/replay, and network game coordination. Bridges core gameplay systems with shell/platform-specific implementations.

## Core Responsibilities
- Game state initialization and transitions (menus, gameplay, dialogs, demos)
- Interface display and user input handling (menus, buttons, clicks)
- Shape/texture collection lifecycle (loading, unloading, querying metadata)
- Game load/save/revert operations
- Input processing (keyboard control, action flags, recording/replay)
- Network multiplayer setup and microphone handling
- Configuration parsing (physics, shapes, sounds, maps, themes via XML)
- Player behavior modifiers and field-of-view management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_information_data` | struct | Metadata for a shape: mirror flags, min light intensity, world bounds, hotpoint |
| `shape_animation_data` | struct | Animation frame/sound data: view counts, frame timing, transfer modes, sound triggers, loop frame |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `strFILENAMES` | enum | global | Resource filenames (shapes, sounds, preferences, maps, physics, music, images, movies, themes) |
| `strPATHS` | const | global | Path resource constants |
| `strERRORS` | enum | global | Error message IDs (processor, memory, serial number, network, corrupt map, etc.) |
| Animation types enum | enum | global | Shape animation frame counts (`_unanimated`, `_animated1` through `_animated8`) |
| Shape types enum | enum | global | Editor shape categories (wall, floor/ceiling, object, other) |
| Keyboard setups enum | enum | global | Predefined key layouts (standard, left-handed, PowerBook, custom) |
| Controller types enum | enum | global | Game modes (single player, network, demo, replay) |
| Game states enum | enum | global | UI/game states (menus, gameplay, dialogs, quit, level change) |
| Bit masks | `#define` | global | Mirror flags, keypoint obscure bits for shape descriptors |

## Key Functions / Methods

### initialize_game_state
- **Signature:** `void initialize_game_state(void);`
- **Purpose:** Initialize game state machine and memory on startup.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Initializes global game state.
- **Calls:** None visible in header.
- **Notes:** Called during game initialization before any gameplay.

### set_game_state / get_game_state
- **Signature:** `void set_game_state(short new_state);` / `short get_game_state(void);`
- **Purpose:** Transition game state and query current state.
- **Inputs:** `new_state` ΓÇô one of the enum values (`_display_main_menu`, `_game_in_progress`, etc.).
- **Outputs/Return:** Current game state (get).
- **Side effects:** Triggers state-specific logic (menu display, gameplay pause).
- **Calls:** None visible.
- **Notes:** Defines primary game state machine; supports screens, gameplay, dialogs, quit.

### update_interface_display
- **Signature:** `void update_interface_display(void);`
- **Purpose:** Refresh UI and menu rendering each frame.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Redraws HUD, menus, and interface elements.
- **Calls:** None visible.
- **Notes:** Called per frame during `_game_in_progress` and menu states.

### portable_process_screen_click
- **Signature:** `void portable_process_screen_click(short x, short y, bool cheatkeys_down);`
- **Purpose:** Handle mouse clicks in the game window.
- **Inputs:** `x`, `y` ΓÇô screen coordinates; `cheatkeys_down` ΓÇô whether cheat modifier is held.
- **Outputs/Return:** None.
- **Side effects:** May toggle menus, select items, trigger cheat actions.
- **Calls:** None visible.
- **Notes:** Entry point for click input from platform layer.

### load_collections / unload_all_collections
- **Signature:** `void load_collections(bool with_progress_bar, bool is_opengl);` / `void unload_all_collections(void);`
- **Purpose:** Load/unload texture and shape collections from disk; manage graphics memory.
- **Inputs:** `with_progress_bar` ΓÇô show loading UI; `is_opengl` ΓÇô render mode flag.
- **Outputs/Return:** None.
- **Side effects:** I/O, memory allocation/deallocation, graphics driver state change.
- **Calls:** None visible.
- **Notes:** Called on level load and game shutdown.

### get_shape_descriptors
- **Signature:** `short get_shape_descriptors(short shape_type, shape_descriptor *buffer);`
- **Purpose:** Retrieve shape descriptors for a given shape type (wall, floor, object, etc.).
- **Inputs:** `shape_type` ΓÇô enum for category; `buffer` ΓÇô output array.
- **Outputs/Return:** Count of shapes returned.
- **Side effects:** None.
- **Calls:** None visible.
- **Notes:** Used by editor or rendering code to enumerate shapes by category.

### extended_get_shape_bitmap_and_shading_table
- **Signature:** `void extended_get_shape_bitmap_and_shading_table(short collection_code, short low_level_shape_index, struct bitmap_definition **bitmap, void **shading_tables, short shading_mode);`
- **Purpose:** Retrieve bitmap and shading table for a shape, keyed by collection and shape index.
- **Inputs:** `collection_code`, `low_level_shape_index` ΓÇô shape identifier; `shading_mode` ΓÇô rendering mode.
- **Outputs/Return:** Populates `bitmap` and `shading_tables` pointers.
- **Side effects:** None.
- **Calls:** None visible.
- **Notes:** Used by rendering code; has macro wrapper `get_shape_bitmap_and_shading_table()`.

### extended_get_shape_information
- **Signature:** `struct shape_information_data *extended_get_shape_information(short collection_code, short low_level_shape_index);`
- **Purpose:** Return shape metadata (flags, lighting, bounds).
- **Inputs:** `collection_code`, `low_level_shape_index` ΓÇô shape identifier.
- **Outputs/Return:** Pointer to shape info struct.
- **Side effects:** None.
- **Calls:** None visible.
- **Notes:** Has macro wrapper; returns persistent data (do not free).

### get_shape_animation_data
- **Signature:** `struct shape_animation_data *get_shape_animation_data(shape_descriptor texture);`
- **Purpose:** Return animation frame, sound, and timing metadata for a shape.
- **Inputs:** `texture` ΓÇô shape descriptor (collection + shape).
- **Outputs/Return:** Pointer to animation data struct.
- **Side effects:** None.
- **Calls:** None visible.
- **Notes:** Used by animation and sound systems.

### load_game / save_game / revert_game
- **Signature:** `bool load_game(bool use_last_load);` / `bool save_game(void);` / `bool revert_game(void);`
- **Purpose:** Load saved game, save current game, or revert to checkpoint.
- **Inputs:** `use_last_load` ΓÇô use previous savefile or prompt user.
- **Outputs/Return:** Success boolean.
- **Side effects:** I/O, memory reallocation, game state restore/save.
- **Calls:** None visible.
- **Notes:** Invoked from game dialogs and menu.

### process_action_flags / check_recording_replaying
- **Signature:** `void process_action_flags(short player_identifier, const uint32 *action_flags, short count);` / `void check_recording_replaying(void);`
- **Purpose:** Apply player input actions; manage recording/replay mode.
- **Inputs:** `player_identifier`, `action_flags` ΓÇô input bitmap per player.
- **Outputs/Return:** None.
- **Side effects:** Updates player state, records/replays actions.
- **Calls:** None visible.
- **Notes:** Called per frame in multiplayer and replay contexts.

### pause_game / resume_game
- **Signature:** `void pause_game(void);` / `void resume_game(void);`
- **Purpose:** Pause/unpause gameplay.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Halts game updates and renders pause menu.
- **Calls:** None visible.
- **Notes:** Used by preferences, menus, and pause dialog.

### network_gather / network_join
- **Signature:** `bool network_gather(bool inResumingGame);` / `int network_join(void);`
- **Purpose:** Host or join a network multiplayer game.
- **Inputs:** `inResumingGame` ΓÇô whether loading a saved game.
- **Outputs/Return:** Success/mode enum (single/multi/failed).
- **Side effects:** Network I/O, player state initialization.
- **Calls:** None visible.
- **Notes:** Entry points for multiplayer setup dialogs.

### get_default_*_spec functions
- **Signature:** `void get_default_{map,physics,sounds,shapes,theme}_spec(FileSpecifier& File);`
- **Purpose:** Locate bundled game resource files by name or search.
- **Inputs:** Reference to `FileSpecifier` output.
- **Outputs/Return:** Populates `File` with path.
- **Side effects:** File system search.
- **Calls:** None visible.
- **Notes:** Called on startup and level changes.

### Infravision_GetParser / ControlPanels_GetParser
- **Signature:** `XML_ElementParser *Infravision_GetParser(void);` / `XML_ElementParser *ControlPanels_GetParser(void);`
- **Purpose:** Return XML parsers for infravision and control-panel configuration.
- **Inputs:** None.
- **Outputs/Return:** Pointer to parser object (or NULL if unavailable).
- **Side effects:** None.
- **Calls:** None visible.
- **Notes:** Used by XML configuration system.

## Control Flow Notes
**Initialization:** `initialize_game_state()` ΓåÆ platform-specific menu/resource loading ΓåÆ `load_collections()`.

**Main loop:** `idle_game_state()` ΓåÆ state machine checks ΓåÆ if `_game_in_progress`, call `process_action_flags()`, `check_recording_replaying()`, then `update_interface_display()`.

**User interaction:** `portable_process_screen_click()` or keyboard input ΓåÆ `do_menu_item_command()` or gameplay actions.

**Level changes:** `set_change_level_destination()` ΓåÆ `check_level_change()` ΓåÆ load new map/shapes ΓåÆ `load_collections()`.

**Network:** `network_gather()` or `network_join()` ΓåÆ `process_action_flags()` with network-sourced inputs.

**Shutdown:** `unload_all_collections()` ΓåÆ `free_and_unlock_memory()`.

## External Dependencies
- **`shape_descriptors.h`** ΓÇô shape_descriptor typedef, macros (GET_DESCRIPTOR_COLLECTION, BUILD_DESCRIPTOR, etc.), collection enum.
- **`XML_ElementParser.h`** ΓÇô XML parser class for config files.
- **`FileSpecifier`** class ΓÇô file path abstraction (defined elsewhere).
- **Undefined types:** `game_data`, `player_start_data`, `entry_point`, `bitmap_definition`, `rgb_color_value`, `low_level_shape_definition`.
- **Inferred imports:** graphics/shading systems, file I/O layer, network stack, input/HID layer.
