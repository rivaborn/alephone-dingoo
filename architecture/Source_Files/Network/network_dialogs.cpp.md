# Source_Files/Network/network_dialogs.cpp

## File Purpose
Implements network game dialogs for the Aleph One engine: hosting (gather), joining, and setup flows. Integrates LAN service discovery (SSLP), metaserver for internet games, and pre-game chat. Uses an abstract factory pattern with SDL-specific implementations.

## Core Responsibilities
- Orchestrate gather/join game flows via `network_gather()` and `network_join()` entry points
- Manage LAN service discovery announcements (`GathererAvailableAnnouncer`) and searches (`JoinerSeekingGathererAnnouncer`)
- Implement dialog base classes (`GatherDialog`, `JoinDialog`, `SetupNetgameDialog`) with SDL concrete subclasses
- Handle player list updates, color/team assignment, and player status callbacks
- Coordinate metaserver advertising and internet game search
- Provide progress dialogs for long-running operations
- Support pre-game chat (LAN, internet, and metaserver modes)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GathererAvailableAnnouncer` | class | Announces host availability via SSLP for LAN discovery |
| `JoinerSeekingGathererAnnouncer` | class | Searches for available hosts on LAN via SSLP |
| `GatherDialog` | class | Abstract base for host/gather UI; callbacks for player join/drop/change |
| `JoinDialog` | class | Abstract base for join UI; manages search, player selection, chat |
| `SetupNetgameDialog` | class | Abstract base for game configuration (map, difficulty, options) |
| `SdlGatherDialog` | class | SDL/widget-based concrete gather dialog |
| `SdlJoinDialog` | class | SDL/widget-based concrete join dialog |
| `SdlSetupNetgameDialog` | class | SDL/widget-based concrete setup dialog |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gMetaserverClient` | `MetaserverClient*` | global | Singleton for internet metaserver connection/chat |
| `gMetaserverChatHistory` | `ChatHistory` | global | Message history for metaserver chat |
| `gPregameChatHistory` | `ChatHistory` | global | Message history for pregame (LAN) chat |
| `sProgressDialog` | `dialog*` | file-static | Current progress dialog, or NULL |
| `sProgressMessage` | `w_static_text*` | file-static | Message widget in progress dialog |
| `sProgressBar` | `w_progress_bar*` | file-static | Optional progress bar widget |

## Key Functions / Methods

### network_gather
- **Signature:** `bool network_gather(bool inResumingGame)`
- **Purpose:** Main entry point to host a network game. Sets up game parameters, starts gathering, advertises on metaserver if requested, and runs the gather loop.
- **Inputs:** `inResumingGame` (whether resuming a saved game)
- **Outputs/Return:** `true` if gather succeeded and game started; `false` if canceled
- **Side effects:** Creates/manages `gMetaserverClient`, broadcasts on LAN via SSLP, modifies network preferences, calls network engine functions (`NetEnter`, `NetGather`, `NetStart`)
- **Calls:** `network_game_setup()`, `GatherDialog::Create()`, `NetEnter()`, `NetGather()`, `NetStart()`, `GameAvailableMetaserverAnnouncer` ctor/methods
- **Notes:** Handles metaserver login exceptions with user alerts; metaserver announcement starts only after game launch

### network_join
- **Signature:** `int network_join(void)`
- **Purpose:** Main entry point to join a network game. Searches for available games (LAN/internet) and manages join flow.
- **Inputs:** None
- **Outputs/Return:** Join result code (`kNetworkJoinedNewGame`, `kNetworkJoinedResumeGame`, `kNetworkJoinFailedUnjoined`, `kNetworkJoinFailedJoined`)
- **Side effects:** Creates/manages `gMetaserverClient`, calls network engine functions (`NetEnter`, `NetGameJoin`, `NetExit`, `NetCancelJoin`)
- **Calls:** `JoinDialog::Create()`, network engine join functions
- **Notes:** Reads preferences only on failure; writes preferences on successful join

### GatherDialog::GatherNetworkGameByRunning
- **Signature:** `bool GatherNetworkGameByRunning()`
- **Purpose:** Run the gather dialog: configure chat, set callbacks, persist autogather preference, wait for players.
- **Inputs:** None (dialog state already initialized)
- **Outputs/Return:** `true` if game started successfully; `false` if canceled
- **Side effects:** Binds autogather preference, sets network callbacks, clears/attaches chat histories, calls `write_preferences()`
- **Calls:** `Run()` (virtual), metaserver notification adapter setup
- **Notes:** Autogather preference is persisted even if dialog canceled

### GatherDialog::idle
- **Signature:** `void idle()`
- **Purpose:** Called repeatedly during gather dialog: search for joining players, auto-gather if enabled.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates `m_ungathered_players` map, may call `gathered_player()` for autogather
- **Calls:** `MetaserverClient::pumpAll()`, `player_search()`, `gathered_player()`, `NetGetNumberOfPlayers()`
- **Notes:** Auto-gathers up to `MAXIMUM_NUMBER_OF_PLAYERS` if autogather is enabled

### GatherDialog::gathered_player
- **Signature:** `bool gathered_player(const prospective_joiner_info& player)`
- **Purpose:** Accept a joining player into the game.
- **Inputs:** Player info from search
- **Outputs/Return:** `true` if successfully gathered; `false` if gathering failed or player limit reached
- **Side effects:** Calls `NetGatherPlayer()`, removes from `m_ungathered_players`, updates widget
- **Calls:** `NetGatherPlayer()`, `update_ungathered_widget()`
- **Notes:** Calls `reassign_player_colors()` callback during gather

### JoinDialog::JoinNetworkGameByRunning
- **Signature:** `const int JoinNetworkGameByRunning()`
- **Purpose:** Run the join dialog: configure UI, bind preferences, wait for join completion.
- **Inputs:** None (dialog state already initialized)
- **Outputs/Return:** Join result code
- **Side effects:** Binds name/color/team preferences to widgets, calls `Run()` (virtual), migrates preferences back
- **Calls:** `Run()` (virtual), preference binders, `getpstr()` for welcome message
- **Notes:** Preference migration happens before and after dialog run to sync UI Γåö prefs

### JoinDialog::gathererSearch
- **Signature:** `void gathererSearch()`
- **Purpose:** Called repeatedly during join dialog: pump network, handle join state changes.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates `join_result`, may call `Stop()`, activates team/color selectors on gather, enables chat
- **Calls:** `JoinerSeekingGathererAnnouncer::pump()`, `MetaserverClient::pumpAll()`, `NetUpdateJoinState()`, `Stop()`
- **Notes:** `NetUpdateJoinState()` returns pseudo-states like `netPlayerAdded`; handles both new-game and resume-game starts

### open_progress_dialog / set_progress_dialog_message / close_progress_dialog / draw_progress_bar
- **Signature:** `void open_progress_dialog(size_t message_id, bool show_progress_bar)`, etc.
- **Purpose:** Manage a non-blocking progress dialog for long operations (map downloads, etc).
- **Inputs:** String resource ID (for message), bool (show bar), or progress values
- **Outputs/Return:** None
- **Side effects:** Creates/updates/deletes global `sProgressDialog`, `sProgressMessage`, `sProgressBar`
- **Calls:** `TS_GetCString()`, dialog manipulation functions
- **Notes:** Dialog started with `start(false)` (non-blocking); no assertions on double-open

## Control Flow Notes
- **Initialization:** `network_gather()` and `network_join()` are the top-level entry points, called by shell when user selects gather or join mode.
- **Gather flow:** `NetEnter()` ΓåÆ `NetGather()` ΓåÆ `GatherDialog::Run()` (idle loop with player search) ΓåÆ `NetStart()` ΓåÆ game.
- **Join flow:** `NetEnter()` ΓåÆ `JoinDialog::Run()` (gathererSearch loop) ΓåÆ game or `NetExit()`.
- **Metaserver:** Connected on-demand; announcement starts after game launch; chat routes through metaserver or pregame history.
- **Dialog classes:** Use abstract factory (`Create()` static method) to instantiate SDL-specific subclasses at link time, avoiding Mac-specific code.

## External Dependencies
- **Network engine:** `NetEnter()`, `NetGather()`, `NetStart()`, `NetGameJoin()`, `NetUpdateJoinState()`, etc. (defined elsewhere)
- **Preferences:** `network_preferences`, `player_preferences`, `serial_preferences` (global structs)
- **Metaserver:** `MetaserverClient`, `GameAvailableMetaserverAnnouncer` (external classes)
- **LAN discovery:** `SSLP_Allow_Service_Discovery()`, `SSLP_Locate_Service_Instances()`, `SSLP_Pump()` (SSLP_API.h)
- **Widgets/Dialog:** `w_title`, `w_toggle`, `w_button`, `dialog`, etc. (widget system, defined elsewhere)
- **Utilities:** `getpstr()`, `pstring_to_string()`, `copy_string_to_cstring()`, `alert_user()` (defined elsewhere)
- **Game:** `reassign_player_colors()`, `MAXIMUM_NUMBER_OF_PLAYERS`, `game_info`, `player_info` (defined in player.h, network.h, etc.)
