# Source_Files/Network/network_messages.h

## File Purpose
Defines message types and serializable message classes for Aleph One's TCPMess network protocol. Enables structured communication during multiplayer game setup, including player joins, capabilities exchange, map/physics/Lua distribution, chat, and connection state management.

## Core Responsibilities
- Define message type IDs (enum constants: kHELLO_MESSAGE through kNETWORK_STATS_MESSAGE)
- Provide reusable templates (`TemplatizedSimpleMessage`, `TemplatizedDataMessage`) for common message patterns
- Implement concrete message classes (HelloMessage, JoinerInfoMessage, TopologyMessage, etc.) with serialization
- Handle large binary payloads (maps, physics, Lua scripts) with optional compression via `BigChunkOfZippedDataMessage`
- Manage client state machine during connection handshake in the `Client` struct
- Dispatch and handle incoming messages via message handlers and a `MessageDispatcher`

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Message type enum (kHELLO_MESSAGE, etc.) | enum constants | Protocol message identifiers (700ΓÇô719) |
| TemplatizedSimpleMessage | template class | Type-safe wrapper for single-value messages (e.g., `JoinPlayerMessage`) |
| AcceptJoinMessage | class | Player join acceptance with NetPlayer data |
| CapabilitiesMessage | class | Feature/version capability exchange |
| ChangeColorsMessage | class | Player color and team assignment |
| ClientInfoMessage | class | Chat client metadata (add/update/remove actions) |
| HelloMessage | class | Version handshake |
| JoinerInfoMessage | class | Prospective joiner metadata and version |
| BigChunkOfZippedDataMessage | class | Large binary payload with zlib compression on deflate |
| TemplatizedDataMessage | template class | Wrapper for bulk data messages (maps, physics, Lua) |
| MapMessage / ZippedMapMessage | typedef | Map data distribution |
| PhysicsMessage / ZippedPhysicsMessage | typedef | Physics data distribution |
| LuaMessage / ZippedLuaMessage | typedef | Lua script data distribution |
| NetworkChatMessage | class | In-game chat with target filtering (players/team/client) |
| NetworkStatsMessage | class | Network telemetry (latency, jitter, errors) |
| ServerWarningMessage | class | Server warnings (e.g., ungatherable joiner) |
| TopologyMessage | class | Network topology (player list, game/player data) |
| Client | struct | Client connection state machine with message dispatch handlers |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Client::check_player | CheckPlayerProcPtr (static member) | static | Callback function pointer for validating joiners |

## Key Functions / Methods

### TemplatizedSimpleMessage<tMessageType, tValueType>::TemplatizedSimpleMessage (constructors)
- Signature: `TemplatizedSimpleMessage()`, `TemplatizedSimpleMessage(tValueType inValue)`
- Purpose: Construct a typed, simple message with optional initial value
- Inputs: Optional value of template type `tValueType`
- Outputs/Return: Message object ready for inflation/deflation
- Side effects: None (pure construction)
- Calls: Parent class `SimpleMessage<tValueType>` constructor
- Notes: Encodes `tMessageType` as compile-time constant in enum `kType`

### AcceptJoinMessage::accepted, player (getters/setters)
- Signature: `bool accepted()`, `void accepted(bool)`, `NetPlayer* player()`, `void player(const NetPlayer*)`
- Purpose: Access and modify join acceptance status and player data
- Inputs: Boolean acceptance flag or NetPlayer reference
- Outputs/Return: Current acceptance state or player pointer
- Side effects: None (accessors only)
- Notes: Player is stored by value, not reference

### CapabilitiesMessage::capabilities
- Signature: `const Capabilities* capabilities()`
- Purpose: Access negotiated feature capabilities
- Outputs/Return: Pointer to internal Capabilities map (stringΓåÆuint32 version pairs)
- Notes: Capabilities define protocol support (kGameworld, kStar, kRing, kLua, etc.)

### NetworkChatMessage (constructor & accessors)
- Signature: `NetworkChatMessage(const char* chatText, int16 senderID, int16 target, int16 targetID)`
- Purpose: Construct a routable chat message
- Inputs: Message text (Γëñ1024 chars), sender ID, target type (kTargetPlayers/kTargetTeam/kTargetPlayer/kTargetClients/kTargetClient), target recipient ID
- Outputs/Return: Accessors return immutable text, sender, target info
- Side effects: Truncates oversized text to CHAT_MESSAGE_SIZE with null terminator
- Notes: Target filtering enables directed messaging (team, individual player, or all)

### Client (constructor & state machine)
- Signature: `Client(CommunicationsChannel*)`
- Purpose: Initialize a client connection with message dispatch handlers
- Inputs: Communication channel for this client
- Outputs/Return: Client object in `_connecting` state
- Side effects: Creates message dispatcher and handler instances (auto_ptr); may register callbacks
- Calls: Sets up handlers for JoinerInfo, Capabilities, AcceptJoin, Chat, ChangeColors messages
- Notes: State enum: _connecting ΓåÆ _connected_but_not_yet_shown ΓåÆ _connected ΓåÆ _awaiting_capabilities ΓåÆ _ungatherable/_joiner_didnt_accept ΓåÆ _awaiting_accept_join ΓåÆ _awaiting_map ΓåÆ _ingame/_disconnect

### Client::can_pregame_chat
- Signature: `bool can_pregame_chat()`
- Purpose: Determine if client can send/receive chat before game start
- Outputs/Return: True if in states: _connected, _connected_but_not_yet_shown, _ungatherable, _joiner_didnt_accept, _awaiting_accept_join, _awaiting_map
- Notes: Inline predicate for chat availability logic

### Client::capabilities_indicate_player_is_gatherable
- Signature: `bool capabilities_indicate_player_is_gatherable(bool warn_joiner)`
- Purpose: Validate joiner capabilities against protocol requirements
- Inputs: Boolean flag to control warning output
- Outputs/Return: True if joiner can participate; may invoke `check_player` callback
- Side effects: May emit warnings if warn_joiner=true and validation fails
- Notes: Prevents incompatible clients from joining (e.g., missing kGameworld, kStar, or kRing support)

### Client::drop
- Signature: `void drop()`
- Purpose: Forcibly disconnect a client
- Side effects: Closes or resets the associated CommunicationsChannel

### Message handler methods (handleJoinerInfoMessage, handleCapabilitiesMessage, etc.)
- Signature: `void handle<MessageType>(MessageType*, CommunicationsChannel*)`
- Purpose: Process incoming message and update client state
- Inputs: Typed message object and optional channel reference
- Side effects: Update Client member state; may trigger state transitions or broadcast updates
- Notes: Dispatched by MessageDispatcher when matching message type received

## Control Flow Notes
**Connection handshake sequence** (inferred from state enum and message types):
1. Client joins: `_connecting` ΓåÆ sends `JoinerInfoMessage` (prospective joiner data + version)
2. Gatherer responds: `CapabilitiesMessage` (feature negotiation)
3. Client moves to: `_awaiting_capabilities` ΓåÆ processes capabilities, validates via `capabilities_indicate_player_is_gatherable()`
4. If accepted: `AcceptJoinMessage` ΓåÆ client advances to `_awaiting_map`
5. Map/Physics/Lua data streamed: `MapMessage`, `PhysicsMessage`, `LuaMessage` (possibly zipped)
6. Topology synced: `TopologyMessage` (final player list)
7. Game starts: `_ingame`

**Message serialization flow** (SmallMessageHelper pattern):
- `deflate()` ΓåÆ calls `reallyDeflateTo(AOStream)` to write binary representation
- `inflateFrom(UninflatedMessage)` ΓåÆ calls `reallyInflateFrom(AIStream)` to parse binary

Large bulk messages (maps, physics, Lua) bypass SmallMessageHelper and use BigChunkOfDataMessage directly, with optional zipping on deflate/unzipping on inflate.

## External Dependencies
- **config.h**: Conditional compilation flag `DISABLE_NETWORKING`
- **cseries.h**: Standard types (int16, uint8, uint32) and macros
- **AStream.h**: Serialization streams (AIStream, AOStream) for binary I/O with endianness support
- **Message.h**: Base class `Message`, `SmallMessageHelper`, `BigChunkOfDataMessage`, `UninflatedMessage`, `SimpleMessage`, `DatalessMessage`
- **SDL_net.h**: SDL networking primitives (linked via config.h)
- **network_capabilities.h**: `Capabilities` class (stringΓåÆversion map)
- **network_private.h**: `NetPlayer`, `NetTopology`, `CommunicationsChannel`, `MessageDispatcher`, `MessageHandler`, `ClientChatInfo`, `prospective_joiner_info` structs/classes; error codes and distribution packet types
