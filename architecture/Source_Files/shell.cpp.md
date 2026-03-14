# Source_Files/shell.cpp

## File Purpose
Main entry point and game loop orchestrator for the Aleph One game engine. Handles application initialization, command-line argument parsing, the main event loop, input dispatch, and graceful shutdown across multiple platforms (Windows, Mac, Linux, BeOS, Dingoo).

## Core Responsibilities
- Parse and validate command-line arguments (fullscreen/windowed, OpenGL, audio, joystick, debug modes)
- Initialize application state across all subsystems (directories, SDL, preferences, rendering, audio, game resources)
- Execute the main game loop: poll events, dispatch input, advance game logic, render frame
- Dispatch input events to appropriate handlers (keyboard, mouse, system quit)
- Manage game-specific input (cheats, UI, map controls, F-key bindings)
- Coordinate graceful application shutdown with resource cleanup
- Platform-specific directory resolution for data files and save games

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `screen_mode_data` | struct | Graphics/display settings (resolution, fullscreen, bit depth, gamma) |
| `view_data` | struct | Camera and rendering parameters (FOV, screen dimensions, effects) |
| `entry_point` | struct | Level metadata (number, name) for level selection |
| `SDL_Event` | union | SDL input/system events (keyboard, mouse, window, quit) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | global | Ordered list of directories searched for game data files |
| `local_data_dir` | `DirectorySpecifier` | global | Per-user local data directory (saves, preferences) |
| `preferences_dir`, `saved_games_dir`, `recordings_dir` | `DirectorySpecifier` | global | Subdirectories for specific save types |
| `user_data_path` | `DirectorySpecifier` | global | Dingoo-specific per-game data path |
| `arg_directory` | `std::string` | global | Command-line-specified data directory |
| `option_nogl`, `option_nosound`, `option_nogamma`, `option_debug`, `option_nojoystick`, `insecure_lua` | `bool` | static | Command-line flags disabling features |
| `force_fullscreen`, `force_windowed` | `bool` | static | Display mode overrides |
| `vidmasterStringSetID` | `short` | static | Custom vidmaster oath string set ID (MML-configurable) |
| `CheatsActive` | `bool` | extern | Cheat mode flag (defined in shell_misc.cpp) |

## Key Functions / Methods

### main
- **Signature:** `int main(int argc, char **argv)`
- **Purpose:** Application entry point; parse arguments, initialize subsystems, run game loop.
- **Inputs:** Command-line arguments
- **Outputs/Return:** Exit code (0 for success)
- **Side effects:** Initializes all game systems; calls atexit(shutdown_application); catches and logs exceptions
- **Calls:** `initialize_application()`, `main_event_loop()`
- **Notes:** Banner printed to stdout; exception handler reports unhandled exceptions to stderr and exits with code 1.

### initialize_application
- **Signature:** `static void initialize_application(void)`
- **Purpose:** Bootstrap all engine subsystems: paths, SDL, resources, preferences, rendering, audio, game state.
- **Inputs:** None (reads global state like force_fullscreen, preferences)
- **Outputs/Return:** None (side effects only)
- **Side effects:** Initializes SDL (video, audio, joystick), loads fonts and MML scripts, sets up directory structure, loads graphics/sound preferences, initializes SoundManager, game window, input controller, gamma correction.
- **Calls:** `DirectorySpecifier::CreateDirectory()`, `SDL_Init()`, `initialize_resources()`, `SetupParseTree()`, `LoadBaseMMLScripts()`, `initialize_preferences()`, `SoundManager::instance()->Initialize()`, `alephone::Screen::instance()->Initialize()`, `initialize_marathon()`, `load_environment_from_preferences()`, `initialize_game_state()`, and 10+ other subsystem init functions.
- **Notes:** Platform-conditional directory resolution (Unix, macOS, Windows, BeOS); respects ALEPHONE_DATA environment variable for data search path; retries with alternative SDL video driver on Windows if primary fails.

### main_event_loop
- **Signature:** `static void main_event_loop(void)`
- **Purpose:** Central game loop: pump events, dispatch input, advance game state, maintain frame timing.
- **Inputs:** None (reads global game state)
- **Outputs/Return:** None; loop terminates when game_state becomes _quit_game
- **Side effects:** Processes all pending SDL events; calls `idle_game_state()` and `execute_timer_tasks()` each frame; may yield CPU with `SDL_Delay()` in non-critical states.
- **Calls:** `SDL_GetTicks()`, `SDL_PumpEvents()`, `SDL_PollEvent()`, `process_event()`, `execute_timer_tasks()`, `idle_game_state()`, `SDL_Delay()`
- **Notes:** Event polling rate is 6 Hz (167ms) in gameplay; higher in menus/cinematics. Yields 10ΓÇô30ms of CPU time in non-critical states to reduce busy-waiting. Handles SDL_Delay timing separately from game tick count.

### shutdown_application
- **Signature:** `static void shutdown_application(void)`
- **Purpose:** Graceful cleanup of SDL and system resources (called via atexit()).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Quits SDL video/audio/joystick; restores gamma table.
- **Calls:** `restore_gamma()`, `SDL_Quit()`, `SDLNet_Quit()` (if HAVE_SDL_NET), `TTF_Quit()` (if HAVE_SDL_TTF)
- **Notes:** Recursive shutdown guard prevents double-cleanup on some platforms.

### process_event
- **Signature:** `static void process_event(const SDL_Event &event)`
- **Purpose:** Dispatcher for all SDL events (keyboard, mouse, window, quit).
- **Inputs:** SDL_Event (type, button/key/state data)
- **Outputs/Return:** None
- **Side effects:** Dispatches to `process_game_key()`, `process_screen_click()`, or state-change functions; may hide/show cursor; may trigger redraw.
- **Calls:** `process_game_key()`, `process_screen_click()`, `set_game_state()`, `mouse_scroll()`, `darken_world_window()`, `update_game_window()`, `SDL_GL_SwapBuffers()`
- **Notes:** Handles SDL_MOUSEBUTTONDOWN (wheel scroll, controller toggle), SDL_KEYDOWN, SDL_QUIT, SDL_ACTIVEEVENT (focus loss), SDL_VIDEOEXPOSE (window redraw).

### process_game_key
- **Signature:** `static void process_game_key(const SDL_Event &event)`
- **Purpose:** Dispatch keyboard input based on game state; route to menus, gameplay, or cinematics.
- **Inputs:** SDL_Event with key data
- **Outputs/Return:** None
- **Side effects:** Calls `handle_game_key()` during gameplay; processes menu selection during menu states; toggles fullscreen; pauses/resumes game.
- **Calls:** `handle_game_key()`, `do_menu_item_command()`, `draw_menu_button_for_command()`, `interface_fade_finished()`, `force_game_state_change()`, `stop_interface_fade()`, `display_main_menu()`, `toggle_fullscreen()`
- **Notes:** Platform-conditional: Alt+Meta vs. Cmd+Meta modifier detection (Mac/Windows); Dingoo has alternate key bindings (SDLK_RETURN, SDLK_LSHIFT for menu nav).

### handle_game_key
- **Signature:** `static void handle_game_key(const SDL_Event &event)`
- **Purpose:** Process in-game keyboard controls (volume, zoom, console, F-keys, cheats).
- **Inputs:** SDL_Event with key data
- **Outputs/Return:** None
- **Side effects:** Adjusts volume/gamma; toggles chase cam, tunnel vision, crosshairs, FPS counter; takes screenshot; changes screen mode/resolution; sets game state on invalid input.
- **Calls:** `SoundManager::instance()->AdjustVolumeUp/Down()`, `ChaseCam_*()`, `Crosshairs_*()`, `SetTunnelVision()`, `zoom_overhead_map_*()`, `scroll_inventory()`, `increment/decrement_replay_speed()`, `Console::instance()->*()`, `OGL_ResetTextures()`, `change_screen_mode()`, `render_screen()`, `change_gamma_level()`, `write_preferences()`, `PlayInterfaceButtonSound()`, `dump_screen()`, `walk_player_list()`, `do_menu_item_command()`, `screen_printf()`
- **Notes:** Extensive F-key bindings (F1ΓÇôF12); guards screen-mode changes to preserve fullscreen state; Dingoo overrides map-view zoom to shoulder buttons.

### get_level_number_from_user
- **Signature:** `short get_level_number_from_user(void)`
- **Purpose:** Display a dialog for the user to select a starting level.
- **Inputs:** None (reads loaded levels from entry_point list)
- **Outputs/Return:** Selected level number, or NONE if cancelled
- **Side effects:** Displays a modal dialog with optional custom vidmaster oath; redraws main menu on return.
- **Calls:** `get_entry_points()`, `TS_IsPresent()`, `TS_CountStrings()`, `TS_GetCString()`, `update_game_window()`
- **Notes:** Constructs a dummy level if no levels load; supports custom vidmaster message via MML stringset.

### dump_screen
- **Signature:** `void dump_screen(void)`
- **Purpose:** Save the current screen to a BMP file in local_data_dir.
- **Inputs:** None (reads SDL video surface)
- **Outputs/Return:** None
- **Side effects:** Writes Screenshot_NNNN.bmp file; allocates temporary surface for OpenGL readback.
- **Calls:** `SDL_GetVideoSurface()`, `SDL_SaveBMP()`, `SDL_CreateRGBSurface()`, `glReadPixels()`, `malloc()`, `free()`, `SDL_FreeSurface()`
- **Notes:** Handles both software and OpenGL rendering; flips OpenGL framebuffer (upside-down) when reading pixels.

### LoadBaseMMLScripts
- **Signature:** `void LoadBaseMMLScripts()`
- **Purpose:** Parse and load MML (Marathon Markup Language) scripts from all data directories.
- **Inputs:** None (uses global data_search_path)
- **Outputs/Return:** None
- **Side effects:** Parses XML files in MML/ and Scripts/ subdirectories; populates game configuration via RootParser.
- **Calls:** `XML_Loader_SDL::ParseDirectory()` (via loader.ParseDirectory())
- **Notes:** Called during initialize_application(); MML controls many game parameters (cheats, difficulty, physics, balance).

---

## Control Flow Notes

**Startup:**
1. `main()` parses command-line args, calls `initialize_application()`.
2. `initialize_application()` sets up directories, SDL, resources (fonts, MML, preferences), rendering, audio, game state.
3. Control returns to `main()`, which calls `main_event_loop()`.

**Main Loop:**
- `main_event_loop()` runs until `game_state == _quit_game`.
- Event polling rate is conditional: 6 Hz during gameplay (to save CPU), higher in menus/intro screens.
- Each frame calls `execute_timer_tasks()` (physics tick) and `idle_game_state()` (AI, render, UI updates).
- CPU yield (`SDL_Delay()`) occurs in non-critical states.

**Input Dispatch:**
- `process_event()` routes SDL events.
- Keyboard events go to `process_game_key()` for state-specific dispatch or `handle_game_key()` for in-game controls.
- Mouse clicks go to `process_screen_click()` (UI) or toggle keyboard controller mode (gameplay).

**Shutdown:**
- Game loop exits when state becomes `_quit_game`.
- `shutdown_application()` is invoked via `atexit()` to clean up SDL.

## External Dependencies
- **Rendering:** `render.h`, `OGL_Render.h`, `screen.h`, `screen_drawing.h`, `game_window.h`
- **Game Logic:** `map.h`, `monsters.h`, `player.h`, `items.h`, `weapons.h`, `interface.h`
- **Audio:** `SoundManager.h`, `Music.h`
- **UI:** `interface_menus.h`, `computer_interface.h`, `sdl_dialogs.h`, `sdl_widgets.h`
- **Input:** `mouse.h` (defined elsewhere)
- **Resources:** `FileHandler.h`, `XML_ParseTreeRoot.h`, `XML_Loader_SDL.h`, `resource_manager.h`, `game_wad.h`
- **Platform/System:** `mytm.h`, `Logging.h`, `Console.h`, `network.h`
- **External Libraries:** SDL (video, audio, joystick, image), OpenGL (optional), SDL_net (optional), SDL_ttf (optional)
- **Standard Library:** `<exception>`, `<algorithm>`, `<vector>`, `<ctype.h>`, `<stdlib.h>`, `<string.h>`, `<unistd.h>` (POSIX), `<windows.h>` (Windows-specific)
