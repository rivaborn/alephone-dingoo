# Source_Files/Network/Metaserver/metaserver_messages.h

## File Purpose

Defines message types and classes for metaserver client communication in Aleph One's TCPMess messaging framework. Provides serializable message classes for login, player/game/room list synchronization, chat, and game administration between metaserver clients and servers.

## Core Responsibilities

- Define message type constants (enums) for bidirectional metaserver protocol
- Provide message classes that encapsulate login, authentication, game creation, and list synchronization
- Support serialization/deserialization via AIStream/AOStream operator overloading
- Define auxiliary data structures (GameDescription, RoomDescription, MetaserverPlayerInfo, GameListEntry)
- Manage handoff tokens for secure server-to-room transitions
- Enable chat and private messaging between clients

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `HandoffToken` | typedef (uint8[32]) | 32-byte secure token for room handoff authentication |
| `GameDescription` | struct | Complete game state snapshot (type, time limit, map checksum, player count, scenario ID, etc.) |
| `RoomDescription` | class | Room metadata (ID, player count, game count, type, server address) |
| `MetaserverPlayerInfo` | class | Player entry in metaserver list (ID, name, colors, admin flags, ranking, status) |
| `GameListMessage::GameListEntry` | nested struct | Single game entry with ID, address, port, description, remaining time, host player info |

## Global / File-Static State

None.

## Key Functions / Methods

### Message Type Enums

Defines three categories:
- **ServerΓåÆClient** (kSERVER_*): ROOMLIST, PLAYERLIST, GAMELIST, DENY, SALT, LOGINSUCCESS, SETPLAYERDATA, LIMIT, BROADCAST, ACCEPT, FIND, STATS
- **ClientΓåÆServer** (kCLIENT_*): LOGIN, ROOM_LOGIN, LOGOUT, NAME_TEAM, CREATEGAME, REMOVEGAME, PLAYERMODE, KEY, SYNCGAMES, GAMEPLAYERLIST, GAMESCORE, RESETGAME, STARTGAME, LOCALIZE, FIND, STATS
- **Bidirectional** (kBOTH_*): CHAT, PRIVATE_MESSAGE, KEEP_ALIVE

### LoginAndPlayerInfoMessage

- **Signature:** Constructor takes `(const std::string& userName, const std::string& playerName, const std::string& teamName)`
- **Purpose:** Client sends initial login credentials and player info to metaserver
- **Inputs:** Username, player name, team name
- **Outputs/Return:** Message object for transmission
- **Side effects:** None (client-to-server only; `reallyInflateFrom()` asserts false)
- **Calls:** Inherits from SmallMessageHelper
- **Notes:** One-way message; server cannot receive this type

### SaltMessage

- **Signature:** No constructor; data populated via `reallyInflateFrom(AIStream&)`
- **Purpose:** Server sends encryption salt and type to client for password hashing
- **Inputs:** AIStream containing `uint16 encryptionType` + 16-byte salt
- **Outputs/Return:** Query via `encryptionType()`, `salt()`
- **Side effects:** None (server-to-client only; `reallyDeflateTo()` asserts false)
- **Calls:** Deserializes from stream
- **Notes:** Supports kPlaintextEncryption and kBraindeadSimpleEncryption types

### GameDescription

- **Signature:** Default constructor initializes with defaults (8 max players, "Untitled Game", scenario from `Scenario::instance()`)
- **Purpose:** Encapsulates complete game state for list transmission
- **Inputs:** 17 fields (type, time limit, checksum, difficulty, max/current players, flags, name, map, scenario ID, version, etc.)
- **Outputs/Return:** Serializable via AIStream >> / AOStream << operators
- **Side effects:** None (pure data container)
- **Calls:** Accesses `Scenario::instance()->GetID()`, `GetName()`, `GetVersion()`
- **Notes:** Includes metadata (scenario ID, protocol version, Aleph One build string) for compatibility checking

### RoomListMessage

- **Signature:** No constructor parameters
- **Purpose:** Server sends list of available rooms to client
- **Inputs:** AIStream containing vector of RoomDescription
- **Outputs/Return:** Access via `rooms()` const getter
- **Side effects:** None (server-to-client only)
- **Calls:** `reallyInflateFrom()` deserializes room vector

### PlayerListMessage

- **Signature:** No constructor parameters
- **Purpose:** Server sends sorted list of online players to client
- **Inputs:** AIStream containing vector of MetaserverPlayerInfo
- **Outputs/Return:** Access via `players()` const getter
- **Side effects:** None (server-to-client only)
- **Calls:** `reallyInflateFrom()` deserializes player vector

### GameListMessage & GameListEntry

- **Signature:** GameListEntry provides ID interface (`id()`, `IdNone = 0xffffffff`)
- **Purpose:** Server sends live game list with compatibility checks and time tracking
- **Inputs:** AIStream containing vector of GameListEntry structs
- **Outputs/Return:** Access via `entries()` const getter; entries provide `compatible()`, `running()`, `minutes_remaining()`, `format_for_chat()`
- **Side effects:** None (server-to-client only)
- **Calls:** `Scenario::instance()->IsCompatible(m_description.m_scenarioID)`, `SDL_GetTicks()` (for time tracking)
- **Notes:** Entries track SDL ticks at last update; `minutes_remaining()` computes time delta; entries are sorted by compatibility/running status/ID

### ChatMessage & PrivateMessage

- **Signature:** `ChatMessage(uint32 inSenderID, const std::string& inSenderName, const std::string& inMessage)`
- **Purpose:** Bidirectional player-to-player and broadcast chat
- **Inputs:** Sender ID, sender name, message text
- **Outputs/Return:** Query via `senderID()`, `senderName()`, `message()`, `directed()` flag
- **Side effects:** Serializes color data (RGB uint16[3])
- **Calls:** `reallyDeflateTo()` / `reallyInflateFrom()`
- **Notes:** kDirectedBit flag (0x1) indicates directed vs. broadcast; PrivateMessage additionally tracks `selectedID`

### MetaserverPlayerInfo

- **Signature:** Constructor takes `AIStream& fromStream` (deserialize-only)
- **Purpose:** Represents single player in metaserver list with admin status and sorting
- **Inputs:** AIStream (deserializes verb, adminFlags, ranking, ID, room ID, rank, colors, name, team, icon, status)
- **Outputs/Return:** Query via `playerID()`, `name()`, `color()`, `team_color()`, `away()`, `verb()`, `target()` flag
- **Side effects:** None
- **Calls:** None
- **Notes:** Static `sort()` comparator orders by admin flags (descending), then status, then ID; admin flags: kNotAdmin (0x0), kBungie (0x1), kAdmin (0x4)

### CreateGameMessage

- **Signature:** `CreateGameMessage(uint16 gamePort, const GameDescription& description)`
- **Purpose:** Client notifies metaserver of new game hosted on specified port
- **Inputs:** Port number and complete GameDescription
- **Outputs/Return:** Message object for transmission
- **Side effects:** None (client-to-server only)
- **Calls:** `reallyDeflateTo()` serializes port and description

### LoginSuccessfulMessage

- **Signature:** No constructor; populated via `reallyInflateFrom(AIStream&)`
- **Purpose:** Server confirms login and hands off client to room server
- **Inputs:** AIStream containing userID and 32-byte handoff token
- **Outputs/Return:** Query via `userID()`, `token()`
- **Side effects:** None (server-to-client only)
- **Calls:** Deserializes from stream

### StartGameMessage

- **Signature:** `StartGameMessage(int32 gameTimeInSeconds)`
- **Purpose:** Server signals game start with initial time limit
- **Inputs:** Game duration in seconds
- **Outputs/Return:** Message object for transmission
- **Side effects:** None (client-to-server only)
- **Calls:** `reallyDeflateTo()` serializes time

### DenialMessage, BroadcastMessage, IDAndLimitMessage

- **Purpose:** DenialMessage (server denies login with code + text), BroadcastMessage (server sends broadcast text), IDAndLimitMessage (server assigns player ID and connection limit)
- **Pattern:** All are server-to-client only; assert false on deflation; deserialize specific fields on inflation

## Control Flow Notes

This file defines the metaserver **login and synchronization phase**:

1. **Login flow:** Client sends LOGIN ΓåÆ Server responds with SALT ΓåÆ Client re-authenticates ΓåÆ Server sends LOGINSUCCESS + handoff token
2. **Room listing:** Server sends ROOMLIST to help client choose
3. **List sync:** Server periodically sends PLAYERLIST, GAMELIST, and game updates (GAMEPLAYERLIST, GAMESCORE)
4. **Game creation:** Client sends CREATEGAME with GameDescription; server sends ACCEPT or DENY
5. **Chat/messaging:** Bidirectional CHAT and PRIVATE_MESSAGE
6. **Keepalive:** KEEP_ALIVE prevents timeout

Fits into **pre-game initialization**; occurs before the client joins an actual game room (which would use different message types).

## External Dependencies

- **Message.h:** Base classes `Message`, `SmallMessageHelper` (templated serialization), `DatalessMessage<T>` (zero-payload messages)
- **AStream.h:** Serialization/deserialization streams `AIStream`, `AOStream` with big-endian/little-endian variants
- **SDL_net.h:** IPaddress struct for room server addresses
- **Scenario.h:** `Scenario::instance()` singleton for game scenario metadata (ID, name, version, compatibility checks)
- **network.h:** `kNetworkSetupProtocolID` constant ("Aleph One WonderNAT V1")
- **Boost.Algorithm:** `to_lower_copy()` for case-insensitive string handling
- **Standard library:** `<string>`, `<vector>` for dynamic data
