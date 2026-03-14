# Source_Files/Network/Metaserver/SdlMetaserverClientUi.cpp

## File Purpose
Implements an SDL-based UI for browsing and joining network games on the Aleph One metaserver. Provides a dialog showing available games, players in rooms, and chat messaging.

## Core Responsibilities
- Constructs and manages the metaserver browser dialog layout (games list, players list, chat)
- Handles periodic updates to game/player lists via a pump function
- Detects and responds to connection loss to the metaserver
- Displays detailed game information on request
- Facilitates game selection and join requests via callbacks

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SdlMetaserverClientUi` | class | SDL specialization of `MetaserverClientUi`; manages dialog construction and event handling |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gMetaserverClient` | `MetaserverClient*` | global (external) | Pointer to the active metaserver client instance |

## Key Functions / Methods

### SdlMetaserverClientUi (constructor)
- **Signature:** `SdlMetaserverClientUi()`
- **Purpose:** Builds the complete metaserver UI dialog, laying out games list, players list, chat history, and control buttons.
- **Inputs:** None
- **Outputs/Return:** Constructed object with dialog `d` initialized and ready to display.
- **Side effects:** Allocates and configures widget hierarchy; wraps widgets in higher-level wrapper classes (e.g., `GameListWidget`, `PlayerListWidget`, `EditTextWidget`).
- **Calls:** Widget constructors (`w_title`, `w_games_in_room`, `w_players_in_room`, `w_colorful_chat`, `w_chat_entry`, `w_tiny_button`, etc.); placer methods to arrange widgets.
- **Notes:** Uses boost::bind to connect game selection callback; activates the chat entry field as initial focus.

### Run()
- **Signature:** `int Run()`
- **Purpose:** Runs the dialog modally, processing events until closed.
- **Inputs:** None
- **Outputs/Return:** Dialog result code (ΓêÆ1 on cancel, otherwise depends on button clicked).
- **Side effects:** Sets up the pump processing function; displays dialog and runs event loop.
- **Calls:** `dialog::set_processing_function()`, `dialog::run()`.
- **Notes:** On result ΓêÆ1, clears `m_joinAddress` to signal cancellation.

### Stop()
- **Signature:** `void Stop()`
- **Purpose:** Closes/dismisses the dialog.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `dialog_ok()` to mark dialog as finished.

### InfoClicked()
- **Signature:** `void InfoClicked()`
- **Purpose:** Displays a detailed information dialog for the currently selected game.
- **Inputs:** None (reads from `gMetaserverClient->game_target()` and metaserver state).
- **Outputs/Return:** None
- **Side effects:** Creates a modal info dialog with tables showing Gatherer info, scenario details, game settings (difficulty, type, limits), and game options (aliens, teams, motion sensor, etc.). May call `JoinGame()` if user clicks JOIN in the info dialog.
- **Calls:** `gMetaserverClient->find_game()`, `gMetaserverClient->find_player()`, `TS_GetCString()` for localized string lookups, `Scenario::instance()->IsCompatible()`, `info_dialog.run()`.
- **Notes:** Populates table with conditional logic for difficulty/game-type string lookup; handles kill limit vs. time limit display based on game options.

### pump()
- **Signature:** `void pump(dialog* d)` (private)
- **Purpose:** Called periodically while the dialog is running to refresh the games list and monitor connection status.
- **Inputs:** Pointer to the active dialog.
- **Outputs/Return:** None
- **Side effects:** Refreshes the games list every 5 seconds; checks if metaserver is still connected; displays alert and stops dialog if disconnected.
- **Calls:** `SDL_GetTicks()`, `games_in_room_w->refresh()`, `gMetaserverClient->isConnected()`, `MetaserverClient::pumpAll()`, `alert_user()`, `Stop()`.
- **Notes:** Uses static `last_update` to throttle refresh rate; sets `m_disconnected` flag to prevent duplicate disconnection alerts.

## Control Flow Notes
- **Initialization:** Constructor builds the dialog layout synchronously.
- **Execution:** `Run()` enters event loop via `dialog::run()`, which periodically invokes `pump()` during idle time.
- **Frame-like behavior:** `pump()` acts as a per-frame update: it polls connection status and refreshes UI every 5 seconds.
- **Shutdown:** Dialog exits on user action (clicking JOIN, CANCEL, etc.) or connection loss.

## External Dependencies
- `sdl_dialogs.h`, `sdl_widgets.h`, `sdl_fonts.h` ΓÇö SDL UI framework
- `network_metaserver.h` ΓÇö metaserver client API
- `network_dialog_widgets_sdl.h` ΓÇö `w_games_in_room`, `w_players_in_room` widget definitions
- `interface.h` ΓÇö `set_drawing_clip_rectangle()` for text clipping
- `metaserver_dialogs.h` ΓÇö likely dialog utilities (e.g., `dialog_ok`, `dialog_cancel`, `alert_user`)
- `TextStrings.h` ΓÇö string set lookups (e.g., `TS_GetCString`)
- Boost ΓÇö `boost::function`, `boost::bind` for callback management
- SDL ΓÇö timer (`SDL_GetTicks()`), core event/surface types
