# Source_Files/Network/network.h

## File Purpose
Public API header for Aleph One's network multiplayer subsystem. Defines types, constants, and function prototypes for game gathering, player joining, game synchronization, chat, and real-time data distribution across networked players.

## Core Responsibilities
- Define game configuration and player metadata structures
- Expose network state management (init, gather, join, sync, active, shutdown)
- Provide callback interfaces for game events (player join/drop, chat receipt)
- Publish functions for host/joiner lifecycle (gather, join, start, distribute data)
- Define network type constants and protocol identifiers
- Expose latency/jitter/error telemetry
- Support pre-game and in-game chat distribution

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `game_info` | struct | Game setup: seed, mode, time/kill limits, difficulty, map, network params |
| `player_info` | struct | Player metadata: name, color, team, serial number |
| `prospective_joiner_info` | struct | Joining player state: stream ID, name, color, team, gathering flag |
| `GatherCallbacks` | abstract class | Virtual interface: `JoinSucceeded()`, `JoiningPlayerDropped()`, `JoinedPlayerDropped()`, `JoinedPlayerChanged()` |
| `ChatCallbacks` | abstract class | Virtual interface: `SendChatMessage()`, `ReceivedMessageFromPlayer()` |
| `InGameChatCallbacks` | class | Singleton impl of `ChatCallbacks`; provides in-game chat UI |
| `NetworkStats` | struct | Telemetry: latency (ms), jitter, error count |

## Global / File-Static State
None (definitions are in network.c).

## Key Functions / Methods

### NetEnter / NetExit
- **Purpose:** Initialize/shut down the network subsystem.
- **Signature:** `bool NetEnter()`, `void NetExit()`
- **Calls:** Initializes state to `netUninitialized` or `netDown`.

### NetGather
- **Signature:** `bool NetGather(void *game_data, short game_data_size, void *player_data, short player_data_size, bool resuming_game)`
- **Purpose:** Host a new game; await joining players.
- **Inputs:** Game config data, player data, resume flag.
- **Outputs/Return:** True if gather started.
- **Side effects:** Network state ΓåÆ `netGathering`; spawns gathering loop.

### NetGameJoin
- **Signature:** `bool NetGameJoin(void *player_data, short player_data_size, const char* host_address_string)`
- **Purpose:** Connect to a gathering game as a joiner.
- **Inputs:** Local player data, host address string.
- **Outputs/Return:** True if join initiated.
- **Side effects:** Network state ΓåÆ `netJoining`; connects to remote gatherer.

### NetStart
- **Signature:** `bool NetStart()`
- **Purpose:** Signal all players ready; transition to active gameplay.
- **Outputs/Return:** True if all players synchronized.
- **Side effects:** Network state ΓåÆ `netStartingUp` ΓåÆ `netActive`.

### NetUpdateJoinState
- **Signature:** `short NetUpdateJoinState()`
- **Purpose:** Poll for state changes during gathering/joining phase.
- **Outputs/Return:** Current or pseudo-state (`netPlayerAdded`, `netChatMessageReceived`, etc.).
- **Notes:** Pseudo-states are transient; calling again returns actual state.

### NetSync / NetUnSync
- **Signature:** `bool NetSync()`, `bool NetUnSync()`
- **Purpose:** Synchronize game clock across all players; called each frame.
- **Outputs/Return:** True if sync successful.

### NetDistributeInformation
- **Signature:** `void NetDistributeInformation(short type, void *buffer, short buffer_size, bool send_to_self, bool send_only_to_team)`
- **Purpose:** Send application-defined data to other players (or team).
- **Inputs:** Message type ID, buffer, size, flags.
- **Side effects:** Queues for transmission to remote players.
- **Calls:** User-registered `NetDistributionProc` callbacks on remote clients.

### NetGetStats
- **Signature:** `const NetworkStats& NetGetStats(int player_index)`
- **Purpose:** Query latency, jitter, error count for a player.
- **Outputs/Return:** Reference to `NetworkStats` struct (latency in ms, or invalid/disconnected sentinel).

### NetGetMostRecentChatMessage / NetDistributeChatMessage
- **Purpose:** Retrieve pending chat and broadcast chat to all players.
- **Signatures:** `bool NetGetMostRecentChatMessage(player_info** outSendingPlayerData, char** outMessage)`, `OSErr NetDistributeChatMessage(short sender_identifier, const char* message)`
- **Notes:** Chat data valid only until next `NetUpdateJoinState()` call.

### Utility Queries
- `NetGetLocalPlayerIndex()`, `NetGetPlayerIdentifier()`, `NetGetNumberOfPlayers()`, `NetGetPlayerData()`, `NetGetGameData()` ΓÇö accessors for topology.
- `NetGetLatency()` ΓÇö local latency estimate (ms).
- `NetGetUnconfirmedActionFlagsCount()` / `NetGetUnconfirmedActionFlag()` ΓÇö prediction buffer for smooth client-side movement.

## Control Flow Notes
**Network state machine** (from enum):
1. `netUninitialized` (initial)
2. **As host:** `NetGather()` ΓåÆ `netGathering` ΓåÆ `netWaiting` (after all join) ΓåÆ `NetStart()` ΓåÆ `netStartingUp` ΓåÆ `netActive`
3. **As joiner:** `NetGameJoin()` ΓåÆ `netJoining` ΓåÆ `netWaiting` (accepted) ΓåÆ `netStartingUp` ΓåÆ `netActive`
4. Pseudo-states (transient returns from `NetUpdateJoinState()`): `netPlayerAdded`, `netPlayerDropped`, `netChatMessageReceived`, `netStartingResumeGame`
5. Shutdown: `netComingDown` ΓåÆ `netDown`

**Frame loop integration** (inferred):
- Pregame: call `NetUpdateJoinState()` repeatedly to await players and chat.
- Ingame: call `NetSync()` each tick, `NetProcessMessagesInGame()` to handle incoming distribution messages, `NetDistributeInformation()` to send state/actions.
- Exit: `NetExit()`.

## External Dependencies
- **Includes:** `"config.h"`, `"cseries.h"`, `"cstypes.h"` (platform abstractions, SDL).
- **Imported:** `struct entry_point` (from map.h), `struct player_start_data` (defined elsewhere), `struct SSLP_ServiceInstance` (service discovery).
- **Callbacks:** Functions like `NetDistributionProc`, `CheckPlayerProcPtr` defined as typedefs; user code registers these.
- **Protocols:** Compile-time constant `kNetworkSetupProtocolID = "Aleph One WonderNAT V1"`; `MARATHON_NETWORK_VERSION` replaced by runtime `get_network_version()` (May 2003 change, implementation in network.c).
