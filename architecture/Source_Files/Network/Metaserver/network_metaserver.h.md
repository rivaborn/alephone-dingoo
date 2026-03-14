# Source_Files/Network/Metaserver/network_metaserver.h

## File Purpose
Provides the client-side API for Aleph One games to connect to and interact with a metaserverΓÇöa central hub managing game lobbies, player lists, chat, and game announcements. Handles connection lifecycle, message routing, and notification callbacks to the UI.

## Core Responsibilities
- Manage connection to metaserver (login, disconnect, reconnection)
- Maintain synchronized lists of rooms, players in room, and active games
- Route incoming messages to handlers (chat, private messages, broadcasts, player/game updates)
- Send game lifecycle events (create, start, reset, delete) and player status changes
- Provide notification callbacks for UI updates via observer pattern
- Manage player ignore lists and targeting (selection state)
- Pump network messages on demand or globally across all instances

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `MetaserverMaintainedList<tElement>` | template class | Generic list container with add/delete/refresh verbs; tracks a "target" (selected) element |
| `MetaserverClient` | class | Main client; manages connection, message handlers, and game/player state |
| `NotificationAdapter` | abstract class | Interface for UI callbacks (players changed, games changed, chat received, etc.) |
| `NotificationAdapterInstaller` | class | RAII guard to temporarily install/restore a notification adapter |
| `LoginDeniedException` | exception | Thrown on login failure with reason codes (bad password, room full, account locked, etc.) |
| `ServerConnectException` | exception | Thrown on network connection failure |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_instances` | `std::set<MetaserverClient*>` | static | All active client instances; used by `pumpAll()` |
| `s_ignoreNames` | `std::set<std::string>` | static | Player names on ignore list; checked at client level |

## Key Functions / Methods

### connect
- Signature: `void connect(const std::string& serverName, uint16 port, const std::string& userName, const std::string& userPassword)`
- Purpose: Establish connection to metaserver, authenticate, and initialize message handlers
- Inputs: Server hostname/IP, port, username, password
- Outputs/Return: (void) Throws `LoginDeniedException` or `ServerConnectException` on failure
- Side effects: Creates `CommunicationsChannel`, `MessageInflater`, `MessageDispatcher`, and message handlers; registers in `s_instances`; may call `NotificationAdapter` callbacks
- Calls: Creates/initializes internal members (`m_channel`, `m_inflater`, `m_dispatcher`, etc.)
- Notes: On `LoginDeniedException`, provides a code enum (SyntaxError, BadUserOrPassword, RoomFull, etc.) for UI handling

### pump
- Signature: `void pump()`
- Purpose: Process one batch of pending network messages and update internal state (player/game lists, etc.)
- Inputs: (none)
- Outputs/Return: (void)
- Side effects: Calls message handlers; updates `m_playersInRoom`, `m_gamesInRoom`, `m_rooms`; may invoke `NotificationAdapter` callbacks
- Calls: Calls `m_dispatcher->processIncomingMessages()` (not visible in header)
- Notes: Called from main game loop; `pumpAll()` iterates all instances

### pumpAll
- Signature: `static void pumpAll()`
- Purpose: Convenience to pump all active metaserver client instances at once
- Inputs: (none)
- Outputs/Return: (void)
- Side effects: Calls `pump()` on each instance in `s_instances`
- Calls: Iterates `s_instances` and calls `pump()` on each
- Notes: Used by main loop to avoid managing client list in game code

### setRoom
- Signature: `void setRoom(const RoomDescription& room)`
- Purpose: Switch the local player to a different room (lobby)
- Inputs: `RoomDescription` struct with room ID, server address, etc.
- Outputs/Return: (void)
- Side effects: Updates `m_room`; sends room login message; triggers refresh of player/game lists for new room
- Calls: Message handlers for room login
- Notes: Implicitly changes `m_playersInRoom` and `m_gamesInRoom` on receipt of new lists from server

### announceGame
- Signature: `void announceGame(uint16 gamePort, const GameDescription& description)`
- Purpose: Advertise a new game instance to the metaserver
- Inputs: Local game port number, `GameDescription` struct with map, difficulty, player limit, etc.
- Outputs/Return: (void)
- Side effects: Sends `CreateGameMessage`; stores `m_gameDescription` and `m_gamePort`
- Calls: Creates and dispatches message
- Notes: Followed later by `announcePlayersInGame()`, `announceGameStarted()`, and eventually `announceGameDeleted()`

### sendChatMessage / sendPrivateMessage
- Signature: `void sendChatMessage(const std::string& message)`; `void sendPrivateMessage(MetaserverPlayerInfo::IDType destination, const std::string& message)`
- Purpose: Broadcast chat or send directed private message
- Inputs: Message text; for private: destination player ID
- Outputs/Return: (void)
- Side effects: Sends message via dispatcher
- Calls: Message creation and dispatch
- Notes: Chat is broadcast to all in room; private messages routed by ID

### player_target / game_target
- Signature: `void player_target(IDType id)`; `IDType player_target()`; similar for games
- Purpose: Set/get which player or game is currently selected (highlighted in UI)
- Inputs: Player/game ID, or none to query
- Outputs/Return: ID or void
- Side effects: Updates `m_target` in `MetaserverMaintainedList`; affects which entry has `target() == true`
- Calls: Delegates to `MetaserverMaintainedList::target()`
- Notes: Used for UI highlighting; `IdNone` (0xffffffff) means no selection

### associateNotificationAdapter / notificationAdapter
- Signature: `void associateNotificationAdapter(NotificationAdapter* adapter)`; `NotificationAdapter* notificationAdapter() const`
- Purpose: Install or retrieve the current UI callback handler
- Inputs: Pointer to adapter (or none to query)
- Outputs/Return: void or adapter pointer
- Side effects: Replaces `m_notificationAdapter`; subsequent updates will call new adapter
- Calls: None (simple pointer assignment)
- Notes: Can be swapped at runtime; `NotificationAdapterInstaller` RAII class manages temporary swaps

## Control Flow Notes

**Initialization (connect phase):**
1. User calls `connect(server, port, user, password)`
2. Opens `CommunicationsChannel`, initiates login handshake
3. Receives `LoginSuccessfulMessage` or `DenialMessage`; throws exception if denied
4. Creates message handlers and installs into dispatcher
5. Adds self to `s_instances`

**Frame/Update (pump phase):**
1. Game calls `pump()` (or `pumpAll()` for all clients)
2. Dispatcher processes queued network messages
3. Handlers route to `handle*Message()` methods
4. Player/game lists updated via `MetaserverMaintainedList::processUpdates()`
5. Notifications triggered via `NotificationAdapter` callbacks (e.g., `playersInRoomChanged()`)

**Game lifecycle:**
1. `announceGame()` ΓÇô publish game to metaserver
2. `announcePlayersInGame(numPlayers)` ΓÇô update player count
3. `announceGameStarted(timeInSeconds)` ΓÇô game begins
4. `announceGameReset()` ΓÇô resets game time
5. `announceGameDeleted()` ΓÇô remove from metaserver listing

**Shutdown:**
1. User calls `disconnect()`
2. Removes self from `s_instances`
3. Tears down channel and handlers

## External Dependencies

- **metaserver_messages.h**: Message types (`ChatMessage`, `PrivateMessage`, `PlayerListMessage`, `GameListMessage`, `RoomDescription`, `GameDescription`, `MetaserverPlayerInfo`)
- **Logging.h**: Logging macros (`logAnomaly1`)
- **CommunicationsChannel, MessageInflater, MessageDispatcher, MessageHandler**: Defined elsewhere; manage socket I/O, decompression, and message routing
- **Standard library**: `<stdexcept>`, `<exception>`, `<vector>`, `<map>`, `<memory>` (auto_ptr), `<set>`
- **config.h**: Build configuration (networking enabled/disabled)
