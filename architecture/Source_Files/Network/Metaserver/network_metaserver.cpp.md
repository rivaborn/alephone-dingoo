# Source_Files/Network/Metaserver/network_metaserver.cpp

## File Purpose
Implements `MetaserverClient`, a client for connecting to and interacting with an Aleph One metaserver. Handles TCP authentication, player/game list management, chat routing, game announcements, and message dispatching using a handler-based architecture.

## Core Responsibilities
- Establish and maintain TCP connections to metaserver and room servers
- Perform login with optional password encryption (plaintext or XOR-based)
- Maintain player and game lists with add/delete/refresh semantics
- Route incoming network messages to appropriate handlers via dispatcher
- Process chat commands (`.available`, `.who`, `.ignore`, `.games`) and regular messages
- Announce game creation, player counts, start/reset/delete events
- Implement user ignore list with persistent muting and guest filtering
- Provide `pump()` mechanism for non-blocking message I/O processing
- Track all active client instances statically for global updates

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MetaserverClient` | class | Main client; manages connection, lists, and message routing |
| `MetaserverMaintainedList<T>` | template class | Generic add/delete/refresh list for players and games (defined in header) |
| `CommunicationsChannel` | class | TCP socket abstraction (external) |
| `MessageDispatcher` | class | Routes messages by type to handlers (external) |
| `MessageInflater` | class | Deserializes wire-format messages (external) |
| `Message` (and derived) | class hierarchy | Protocol message types (external) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MetaserverClient::s_instances` | `set<MetaserverClient*>` | static | Track all live client instances for `pumpAll()` |
| `MetaserverClient::s_ignoreNames` | `set<string>` | static | Global ignore list ("Bacon" is hardcoded in) |
| `kKeyLength` | `const int` | file-static | Password encryption key size (16 bytes) |

## Key Functions / Methods

### MetaserverClient (constructor)
- **Signature:** `MetaserverClient()`
- **Purpose:** Initialize client infrastructure and register message handlers.
- **Inputs:** None.
- **Outputs/Return:** Instance ready for `connect()`.
- **Side effects:** Creates `CommunicationsChannel`, `MessageInflater`, `MessageDispatcher`; registers 9 message handlers; inserts self into `s_instances`; adds "Bacon" to ignore list.
- **Calls:** `newMessageHandlerMethod()` (template), inflaterΓåÆ`learnPrototype()`, dispatcherΓåÆ`setHandlerForType()`.
- **Notes:** Uses `auto_ptr` for RAII; all handlers are bound to this instance.

### connect
- **Signature:** `void connect(const std::string& serverName, uint16 port, const std::string& userName, const std::string& userPassword)`
- **Purpose:** Establish TCP connection to metaserver, authenticate, receive room list, then connect to room server.
- **Inputs:** Server hostname/IP, port, username, password.
- **Outputs/Return:** Throws `LoginDeniedException`, `ServerConnectException`, or `CommunicationsChannel::FailedToReceiveSpecificMessageException` on error.
- **Side effects:** Clears `m_playersInRoom` and `m_gamesInRoom`; stores `m_playerID` and `m_rooms`; switches message handler from `m_loginDispatcher` to `m_dispatcher`.
- **Calls:** `m_channelΓåÆconnect()`, `receiveMessage()`, `receiveSpecificMessageOrThrow()`, password encryption logic, `m_dispatcherΓåÆhandle()`.
- **Notes:** Supports plaintext and XOR-based "braindeadSimple" encryption; selects "Arrival" room preferentially; performs two-stage connection (metaserver ΓåÆ room).

### pump
- **Signature:** `void pump()`
- **Purpose:** Non-blocking I/O and message dispatch for a single frame.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Processes outgoing and incoming queue; notifies adapter of disconnect if channel closes and notification not yet sent.
- **Calls:** `m_channelΓåÆpump()`, `m_channelΓåÆdispatchIncomingMessages()`, adapterΓåÆ`roomDisconnected()`.
- **Notes:** Sets `m_notifiedOfDisconnected` flag to avoid duplicate notifications.

### pumpAll (static)
- **Signature:** `static void pumpAll()`
- **Purpose:** Call `pump()` on all active client instances.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Iterates and calls instance method on each in `s_instances`.
- **Calls:** `std::for_each()` with `mem_fun()`.

### disconnect
- **Signature:** `void disconnect()`
- **Purpose:** Cleanly close connection (called in destructor).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Enqueues logout message, pumps once, closes channel; swallows exceptions.
- **Calls:** `m_channelΓåÆenqueueOutgoingMessage()`, `m_channelΓåÆpump()`, `m_channelΓåÆdisconnect()`.
- **Notes:** No-throw guarantee (try/catch suppresses errors).

### handleChatMessage
- **Signature:** `void handleChatMessage(ChatMessage* message, CommunicationsChannel* inChannel)`
- **Purpose:** Receive and route chat messages; apply muting filters.
- **Inputs:** Deserialized `ChatMessage`.
- **Outputs/Return:** None.
- **Side effects:** Notifies adapter of chat or private message; filters guest users if muted; checks ignore list.
- **Calls:** `remove_formatting()`, `boost::algorithm::starts_with()`, `m_notificationAdapterΓåÆreceivedChatMessage()` or `receivedPrivateMessage()`.
- **Notes:** Strips special character `\260` (octal); only processes `internalType() == 0`.

### handlePrivateMessage
- **Signature:** `void handlePrivateMessage(PrivateMessage* message, CommunicationsChannel* inChannel)`
- **Purpose:** Receive private messages; apply same muting and ignore filtering.
- **Inputs:** Deserialized `PrivateMessage`.
- **Outputs/Return:** None.
- **Side effects:** Same as `handleChatMessage`.
- **Calls:** Same filtering and notification calls.

### handlePlayerListMessage
- **Signature:** `void handlePlayerListMessage(PlayerListMessage* inMessage, CommunicationsChannel* inChannel)`
- **Purpose:** Update player list and notify UI.
- **Inputs:** `PlayerListMessage` with add/delete/refresh updates.
- **Outputs/Return:** None.
- **Side effects:** Dedups invisible players leaving; processes updates via `m_playersInRoom.processUpdates()`; notifies adapter.
- **Calls:** `find_player()`, `m_playersInRoom.processUpdates()`, adapterΓåÆ`playersInRoomChanged()`.
- **Notes:** Filters out delete verbs for players not in list (invisible departures).

### handleGameListMessage
- **Signature:** `void handleGameListMessage(GameListMessage* inMessage, CommunicationsChannel* inChannel)`
- **Purpose:** Update game list and enrich with host player names.
- **Inputs:** `GameListMessage` with game entries.
- **Outputs/Return:** None.
- **Side effects:** Enriches each entry with host name from `m_playersInRoom`; processes via `m_gamesInRoom.processUpdates()`; notifies adapter.
- **Calls:** `m_playersInRoom.find()`, `m_gamesInRoom.processUpdates()`, adapterΓåÆ`gamesInRoomChanged()`.

### sendChatMessage
- **Signature:** `void sendChatMessage(const std::string& message)`
- **Purpose:** Send or handle special commands prefixed with `.`.
- **Inputs:** Message string.
- **Outputs/Return:** None.
- **Side effects:** Queues `ChatMessage` or routes to command handler; notifies adapter of local message for commands.
- **Calls:** Adapter notification methods; `m_channelΓåÆenqueueOutgoingMessage()`.
- **Notes:** Special commands: `.available`, `.who`, `.pork` (Easter egg), `.ignore`, `.ignore <name>`, `.games`.

### announceGame
- **Signature:** `void announceGame(uint16 gamePort, const GameDescription& description)`
- **Purpose:** Register a hosted game on the metaserver.
- **Inputs:** Port and game metadata.
- **Outputs/Return:** None.
- **Side effects:** Stores `m_gameDescription` and `m_gamePort`; sets `m_gameAnnounced = true`; enqueues `CreateGameMessage`.
- **Calls:** `m_channelΓåÆenqueueOutgoingMessage()`.

### announceGameStarted
- **Signature:** `void announceGameStarted(int32 gameTimeInSeconds)`
- **Purpose:** Mark game as running and closed to new players.
- **Inputs:** Game time limit (or `INT32_MAX` for unlimited).
- **Outputs/Return:** None.
- **Side effects:** Sets `m_gameDescription.m_closed = true` and `m_running = true`; enqueues `CreateGameMessage` and `StartGameMessage`.
- **Calls:** `m_channelΓåÆenqueueOutgoingMessage()`.

### ignore (by string)
- **Signature:** `void ignore(const std::string& name)`
- **Purpose:** Toggle ignore status for a player by name.
- **Inputs:** Player name (may contain format codes).
- **Outputs/Return:** None.
- **Side effects:** Adds or removes from `s_ignoreNames`; notifies adapter.
- **Calls:** `remove_formatting()`, adapterΓåÆ`receivedLocalMessage()`.
- **Notes:** Removes format codes before lookup.

### ignore (by ID)
- **Signature:** `void ignore(MetaserverPlayerInfo::IDType id)`
- **Purpose:** Toggle ignore status for a player by ID.
- **Inputs:** Player ID.
- **Outputs/Return:** None.
- **Side effects:** Looks up player in list, removes special char, calls string-based ignore.
- **Calls:** `m_playersInRoom.find()`, `ignore(string)`.

### is_ignored
- **Signature:** `bool is_ignored(MetaserverPlayerInfo::IDType id)`
- **Purpose:** Check if a player is on the ignore list.
- **Inputs:** Player ID.
- **Outputs/Return:** Boolean.
- **Side effects:** None.
- **Calls:** `m_playersInRoom.find()`, `remove_formatting()`.

### Trivial accessors and setters
- `isConnected()`, `setPlayerName()`, `setPlayerTeamName()`, `setAway()`, `setMode()`, `syncGames()`: Simple delegates to channel or storage.
- `handleKeepAliveMessage()`: Responds with empty `KeepAliveMessage`.
- `handleBroadcastMessage()`, `handleRoomListMessage()`: Notify or store lists.
- `handleUnexpectedMessage()`: Log anomaly.
- `handleSetPlayerDataMessage()`: Empty (no-op).

## Control Flow Notes

**Initialization phase:**
1. Constructor sets up infrastructure.
2. `connect()` performs metaserver login, receives room list, connects to room server.

**Frame/pump phase:**
- `pump()` (or `pumpAll()` globally) called per frame.
- `m_channelΓåÆpump()` handles I/O.
- `m_channelΓåÆdispatchIncomingMessages()` routes to `m_dispatcher`, which invokes message handlers.
- Handlers update `m_playersInRoom`, `m_gamesInRoom`, notify adapter.

**Shutdown phase:**
- `disconnect()` sends logout, closes channel.
- Destructor calls `disconnect()` and removes self from `s_instances`.

**Game lifecycle:**
- `announceGame()` ΓåÆ server lists game.
- `announcePlayersInGame()` ΓåÆ update player count.
- `announceGameStarted()` ΓåÆ mark running/closed.
- `announceGameReset()` / `announceGameDeleted()` ΓåÆ update status.

## External Dependencies
- **CommunicationsChannel** (defined elsewhere): TCP socket abstraction with message buffering.
- **MessageInflater, MessageDispatcher, MessageHandler** (defined elsewhere): Serialization and message routing.
- **Message and derived classes** (metaserver_messages.h): Protocol message types.
- **MetaserverPlayerInfo, GameListMessage::GameListEntry, RoomDescription** (metaserver_messages.h): Data structures.
- **boost::algorithm::starts_with**: String prefix matching.
- **std::set, std::vector, std::string, std::auto_ptr**: STL containers.
- **Logging.h**: `logAnomaly1()`, `logAnomaly2()` for diagnostics.
- **network_preferences** (external): Preferences object for mute settings.
