# Source_Files/Network/Metaserver/metaserver_dialogs.h

## File Purpose
Defines UI abstractions and handlers for the metaserver client in Aleph One. Provides the main dialog interface for players to browse games, chat, and join servers via the metaserver.

## Core Responsibilities
- Abstract UI layer for metaserver interaction (platform-independent)
- Announce locally-hosted games to the metaserver
- Bridge metaserver notifications (chat, player lists, game updates) to UI widgets
- Manage game/player selection, chat input, and join operations
- Provide factory method for concrete UI implementations (SDL/Carbon determined at link-time)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `GameAvailableMetaserverAnnouncer` | class | Announces a locally-hosted game to the metaserver for a limited time |
| `GlobalMetaserverChatNotificationAdapter` | class | Implements NotificationAdapter to receive metaserver events (chat, players, games, disconnects) |
| `MetaserverClientUi` | class | Abstract base for the metaserver dialog UI; concrete impl determined at link-time |

## Global / File-Static State
None. Comment notes that `gMetaserverClient` is used elsewhere instead of instance member.

## Key Functions / Methods

### run_network_metaserver_ui
- **Signature:** `const IPaddress run_network_metaserver_ui()`
- **Purpose:** Entry point for the metaserver UI; runs the dialog and returns the IP address of a joined game
- **Inputs:** None
- **Outputs/Return:** IP address to connect to, or null if cancelled
- **Side effects:** Displays modal UI; sends/receives metaserver messages; updates global game state
- **Calls:** (defined elsewhere)
- **Notes:** Non-inferable from this header whether blocking or async

### GameAvailableMetaserverAnnouncer::Start
- **Signature:** `void Start(int32 time_limit)`
- **Purpose:** Announce the game to the metaserver for `time_limit` seconds
- **Inputs:** `time_limit` ΓÇô duration in seconds
- **Outputs/Return:** None
- **Side effects:** Sends announcement message to metaserver; maintains presence until timeout
- **Calls:** (uses `gMetaserverClient` elsewhere)
- **Notes:** Must be paired with game creation; comment indicates client is global, not instance member

### MetaserverClientUi::Create
- **Signature:** `static std::auto_ptr<MetaserverClientUi> Create()`
- **Purpose:** Abstract factory; returns platform-specific UI implementation
- **Inputs:** None
- **Outputs/Return:** Auto-ptr to concrete UI subclass (SDL or Carbon)
- **Side effects:** Allocates heap memory
- **Calls:** (implementation provided by linker)
- **Notes:** Concrete type selected at link-time based on platform

### MetaserverClientUi::GetJoinAddressByRunning
- **Signature:** `const IPaddress GetJoinAddressByRunning()`
- **Purpose:** Run the dialog modally; return the address of the selected game
- **Inputs:** None
- **Outputs/Return:** IP address of joined game (or null if cancelled)
- **Side effects:** Displays UI; updates widget state; communicates with metaserver
- **Calls:** `Run()` (virtual; implemented in subclass)
- **Notes:** Used by main UI flow to fetch a joinable server

### Callback methods (Game/Player selection, Chat, Buttons)
- **Methods:** `GameSelected()`, `PlayerSelected()`, `JoinGame()`, `JoinClicked()`, `MuteClicked()`, `sendChat()`, etc.
- **Purpose:** Handle UI interactions (selection, button clicks, text input)
- **Pattern:** Update internal state, dispatch metaserver messages, update widget visibility/state
- **Notes:** Implemented in concrete subclass; virtual overrides for info display vary by platform

### Notification adapter overrides
- **Methods:** `playersInRoomChanged()`, `gamesInRoomChanged()`, `receivedChatMessage()`, `receivedPrivateMessage()`, `roomDisconnected()`
- **Purpose:** Respond to metaserver events by updating displayed widget lists and chat history
- **Notes:** Called by `MetaserverClient` when updates arrive from server

## Control Flow Notes
This file is part of the UI initialization ΓåÆ join flow:
1. `run_network_metaserver_ui()` is called from main menu
2. Concrete `MetaserverClientUi` is created via factory
3. Dialog runs, showing game list and chat
4. Player selects a game ΓåÆ `GetJoinAddressByRunning()` returns IP
5. Main loop connects to that address

The UI subscribes to `MetaserverClient::NotificationAdapter` to receive real-time updates (games, players, chat) while the dialog is open.

## External Dependencies
- **`network_metaserver.h`** ΓÇö `MetaserverClient`, `MetaserverClient::NotificationAdapter`, `MetaserverPlayerInfo`
- **`metaserver_messages.h`** ΓÇö `GameListMessage::GameListEntry`, message types
- **`shared_widgets.h`** ΓÇö `PlayerListWidget`, `GameListWidget`, `EditTextWidget`, `ColorfulChatWidget`, `ButtonWidget`
- **Standard library** ΓÇö `<vector>`, `std::auto_ptr`
