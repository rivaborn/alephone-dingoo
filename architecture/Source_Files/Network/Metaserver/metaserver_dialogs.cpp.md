# Source_Files/Network/Metaserver/metaserver_dialogs.cpp

## File Purpose
Implements the UI dialog system for Aleph One's metaserver client, allowing players to browse available games, chat, and join network games. Handles game announcement, player interaction, and chat notifications.

## Core Responsibilities
- **Metaserver connection setup**: Initialize player credentials and connect to metaserver with optional update checking
- **Game announcement**: Register active games with the metaserver for visibility to other players
- **Dialog lifecycle management**: Create, run, and tear down the main metaserver browsing UI
- **Chat and player notifications**: Translate metaserver events (player joins, chat messages, game availability) into UI updates
- **Game/player selection and joining**: Manage user interactions with game lists, player lists, and chat
- **UI state management**: Toggle button activation states based on current selection and game compatibility

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| GameAvailableMetaserverAnnouncer | class | Wraps game announcement to metaserver with full game description |
| GlobalMetaserverChatNotificationAdapter | class | Adapts metaserver notification callbacks into chat history updates |
| MetaserverClientUi | class | Abstract base for platform-specific UI implementation; manages game/player selection and chat widget callbacks |
| GameDescription | struct (extern) | Contains game type, time limit, difficulty, map name, embedded content flags |
| ColoredChatEntry | struct (implicit) | Chat message with sender name and color information |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| gMetaserverClient | MetaserverClient\* (extern) | global | Global metaserver client instance shared across the application |
| gMetaserverChatHistory | ChatHistory (extern) | global | Global chat history buffer attached to UI widgets |
| user_informed | bool (static) | `setupAndConnectClient()` | Prevents repeated update notifications to user |
| first_check | bool (static) | `setupAndConnectClient()` | Guards first-time update checking with timing windows |

## Key Functions / Methods

### run_network_metaserver_ui
- Signature: `const IPaddress run_network_metaserver_ui()`
- Purpose: Entry point; creates and runs metaserver UI dialog, returns join address
- Outputs/Return: IPaddress (host and port of selected game)
- Calls: `MetaserverClientUi::Create()`, `GetJoinAddressByRunning()`

### setupAndConnectClient
- Signature: `void setupAndConnectClient(MetaserverClient& client)`
- Purpose: Configure player name, check for updates, connect to metaserver server
- Inputs: Reference to MetaserverClient to configure
- Side effects: Sets player name on client; if update checking enabled, polls `Update::instance()` and may display modal progress/update-available dialog; calls `client.connect()`
- Calls: `client.setPlayerName()`, `Update::instance()->GetStatus()`, `open_progress_dialog()`, `close_progress_dialog()`, `client.setPlayerTeamName()`, `client.connect()`
- Notes: Contains static flags `user_informed` and `first_check` to avoid repeated work; waits up to 2.5s for update check with UI feedback

### GameAvailableMetaserverAnnouncer (constructor)
- Signature: `GameAvailableMetaserverAnnouncer(const game_info& info)`
- Purpose: Extract game info and announce to metaserver via global client
- Inputs: game_info struct containing game state
- Side effects: Calls `setupAndConnectClient()`, builds GameDescription, retrieves embedded physics/lua info, calls `gMetaserverClient->announceGame()`
- Calls: `setupAndConnectClient()`, `level_has_embedded_physics_lua()`, `gMetaserverClient->announceGame()`
- Notes: Constructs description from game type, time limit, difficulty, map name, netscript/physics file info

### GameAvailableMetaserverAnnouncer::Start
- Signature: `void Start(int32 time_limit)`
- Purpose: Signal that a previously announced game has started
- Inputs: time_limit in game ticks
- Calls: `gMetaserverClient->announceGameStarted()`

### GlobalMetaserverChatNotificationAdapter::playersInRoomChanged
- Signature: `void playersInRoomChanged(const std::vector<MetaserverPlayerInfo>&)`
- Purpose: Log player join/leave events to chat history
- Side effects: Appends "X has joined/left the room" messages to gMetaserverChatHistory
- Calls: `receivedLocalMessage()`

### GlobalMetaserverChatNotificationAdapter::gamesInRoomChanged
- Signature: `void gamesInRoomChanged(const std::vector<GameListMessage::GameListEntry>&)`
- Purpose: Log game creation/deletion and play notification sound
- Side effects: Appends game announcement to chat; plays "got ball" sound on new non-running game
- Calls: `receivedLocalMessage()`, `PlayInterfaceButtonSound()`

### GlobalMetaserverChatNotificationAdapter::receivedChatMessage / receivedPrivateMessage / receivedBroadcastMessage / receivedLocalMessage / roomDisconnected
- Purpose: Append various message types to chat history with appropriate coloring and formatting
- Side effects: Appends colored entry to gMetaserverChatHistory; may play sound effects
- Calls: `color_entry()`, `gMetaserverChatHistory.append()`, `PlayInterfaceButtonSound()`

### MetaserverClientUi::GetJoinAddressByRunning
- Signature: `const IPaddress GetJoinAddressByRunning()`
- Purpose: Main UI loop; sets up widgets, associates callbacks, runs platform-specific dialog
- Outputs/Return: IPaddress of selected game (zero-initialized if cancelled)
- Side effects: Sets up all widget callbacks using boost::bind; attaches chat history; calls `Run()` (platform-specific); may delete and recreate gMetaserverClient on cancel
- Calls: `setupAndConnectClient()`, `gMetaserverClient->associateNotificationAdapter()`, widget callback binding, `gMetaserverChatHistory.clear()`, `Run()`, `handleCancel()`
- Notes: Asserts `!m_used` (one-shot use); clears `m_joinAddress` at start

### MetaserverClientUi::GameSelected
- Signature: `void GameSelected(GameListMessage::GameListEntry game)`
- Purpose: Handle game selection; supports double-click join and deselection
- Side effects: Sets game_target on client; on double-click with compatible scenario, calls `JoinClicked()`; updates sorted game list in widget; calls `UpdateGameButtons()`
- Notes: Double-click detected via timestamp comparison (333ms threshold); records `m_lastGameSelected`

### MetaserverClientUi::JoinGame
- Signature: `void JoinGame(const GameListMessage::GameListEntry&)`
- Purpose: Extract address/port from game entry, store in m_joinAddress, and exit dialog
- Side effects: Copies game IP and port to m_joinAddress; calls `Stop()`

### MetaserverClientUi::PlayerSelected / MuteClicked / sendChat / ChatTextEntered / handleCancel / UpdatePlayerButtons / UpdateGameButtons
- Notes: Trivial helpers for UI interaction; PlayerSelected toggles selection and updates player list; MuteClicked calls client.ignore(); sendChat routes message as public or private; handleCancel resets client

## Control Flow Notes
- **Init**: `run_network_metaserver_ui()` ΓåÆ `Create()` + `GetJoinAddressByRunning()`
- **Setup**: `setupAndConnectClient()` connects and checks updates; notification adapter installed
- **Loop**: Platform-specific `Run()` dispatches events; user selects games/players, sends chat
- **Callbacks**: Game/player selection, mute, chat text ΓåÆ client method calls
- **Exit**: Game join calls `Stop()`; returns address to caller for connection

## External Dependencies
- **Includes**: network_metaserver.h (MetaserverClient, MetaserverPlayerInfo), metaserver_dialogs.h, network_private.h (GAME_PORT), preferences.h (player/network/environment prefs), alephversion.h (version strings), map.h (_force_unique_teams), SoundManager.h, game_wad.h (level_has_embedded_physics_lua), Update.h, progress.h
- **External symbols**: gMetaserverClient, gMetaserverChatHistory (externs); player_preferences, network_preferences, environment_preferences; Scenario::instance(); PlayInterfaceButtonSound(); level_has_embedded_physics_lua(); Update::instance()
