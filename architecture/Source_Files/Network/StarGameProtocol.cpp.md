# Source_Files/Network/StarGameProtocol.cpp

## File Purpose
Glue layer interfacing the star network protocol implementation with the rest of Aleph One. Bridges game logic with low-level networking code for star-topology (hub-and-spoke) multiplayer synchronization, including packet dispatch, action queue adaptation, and lossy data streaming.

## Core Responsibilities
- Implements `StarGameProtocol` class extending `NetworkGameProtocol` for protocol abstraction
- Manages protocol lifecycle: initialization (`Enter`), synchronization (`Sync`), cleanup (`UnSync`)
- Routes packets between hub and spoke network modules based on local/remote role
- Adapts legacy action queues to tick-based queue interface via `LegacyActionQueueToTickBasedQueueAdapter`
- Distributes lossy streaming data (configurable per distribution type)
- Maintains player connectivity state and marks players as network-dead
- Provides XML configuration parsing and preference serialization

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `LegacyActionQueueToTickBasedQueueAdapter<tValueType>` | template class | Adapts legacy action queues to tick-based interface; templated for action_flags_t |
| `WritableTickBasedActionQueue` | typedef (abstract) | Queue interface for writing tick-indexed action flags (from TickBasedCircularQueue.h) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sStarQueues` | `WritableTickBasedActionQueue*[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | static (file) | Per-player action queues for network players |
| `sHubIsLocal` | `bool` | static (file) | Indicates whether hub role resides on this machine |
| `sTopology` | `NetTopology*` | static (file) | Current network topology (players, addresses, connectivity) |
| `sNetStatePtr` | `short*` | static (file) | Pointer to global network state (netStartingUp, netActive, netDown) |
| `sStarParser` | `XML_ElementParser` | static (file) | XML parser for star protocol configuration section |

## Key Functions / Methods

### StarGameProtocol::Enter
- Signature: `bool Enter(short* inNetStatePtr)`
- Purpose: Initialize the protocol layer with network state pointer
- Inputs: `inNetStatePtr` ΓÇö pointer to global network state variable
- Outputs/Return: `true` (always succeeds)
- Side effects: Caches network state pointer in `sNetStatePtr`
- Calls: (none)
- Notes: Called once at protocol initialization; minimal setup

### StarGameProtocol::Sync
- Signature: `bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex)`
- Purpose: Synchronize network state; initialize hub if local, initialize spoke in all cases
- Inputs:
  - `inTopology` ΓÇö player list, addresses, connectivity flags
  - `inSmallestGameTick` ΓÇö earliest tick to sync from
  - `inLocalPlayerIndex` ΓÇö index of this machine's player
  - `inServerPlayerIndex` ΓÇö index of the hub/server player
- Outputs/Return: `true` on success
- Side effects: 
  - Allocates and caches per-player `LegacyActionQueueToTickBasedQueueAdapter` in `sStarQueues`
  - Sets `sHubIsLocal` based on role comparison
  - Calls `hub_initialize()` if local is server, always calls `spoke_initialize()`
  - Sets `*sNetStatePtr = netActive`
- Calls: `GetRealActionQueues()`, `hub_initialize()`, `spoke_initialize()`
- Notes: Determines role (hub vs spoke); creates adapter queues for connected players only

### StarGameProtocol::UnSync
- Signature: `bool UnSync(bool inGraceful, int32 inSmallestPostgameTick)`
- Purpose: Shut down network synchronization; clean up queues and protocol state
- Inputs:
  - `inGraceful` ΓÇö whether to send disconnect messages
  - `inSmallestPostgameTick` ΓÇö final tick for post-game replay/cleanup
- Outputs/Return: `true` on success
- Side effects:
  - Calls `spoke_cleanup()` and optionally `hub_cleanup()`
  - Deletes all allocated action queue adapters
  - Sets `*sNetStatePtr = netDown`
- Calls: `spoke_cleanup()`, `hub_cleanup()`
- Notes: Idempotent within a sync cycle (checks netState before cleanup)

### StarGameProtocol::DistributeInformation
- Signature: `void DistributeInformation(short type, void *buffer, short buffer_size, bool send_to_self, bool only_send_to_team)`
- Purpose: Send information to other players; uses lossy streaming if distribution type is lossy
- Inputs:
  - `type` ΓÇö distribution message type
  - `buffer` ΓÇö message payload
  - `buffer_size` ΓÇö payload size in bytes
  - `send_to_self` ΓÇö whether sender receives own message
  - `only_send_to_team` ΓÇö restrict to team members
- Outputs/Return: (none)
- Side effects: Queues message in spoke's lossy distribution if configured
- Calls: `NetGetDistributionInfoForType()`, `spoke_distribute_lossy_streaming_bytes_to_everyone()`
- Notes: Only distributes if distribution type is registered as lossy; no-op otherwise

### StarGameProtocol::PacketHandler
- Signature: `void PacketHandler(DDPPacketBufferPtr packet)`
- Purpose: Route incoming network packet to hub or spoke handler based on role
- Inputs: `packet` ΓÇö DDP packet buffer
- Outputs/Return: (none)
- Side effects: Sends packet to appropriate handler (hub or spoke)
- Calls: `hub_received_network_packet()` or `spoke_received_network_packet()`
- Notes: Simple dispatcher; determines role from `sHubIsLocal`

### StarGameProtocol::GetNetTime
- Signature: `int32 GetNetTime()`
- Purpose: Query current network time
- Inputs: (none)
- Outputs/Return: Current network tick from spoke
- Side effects: (none)
- Calls: `spoke_get_net_time()`
- Notes: Delegates to spoke implementation

### StarGameProtocol::GetUnconfirmedActionFlagsCount
- Signature: `int32 GetUnconfirmedActionFlagsCount()`
- Purpose: Count unconfirmed player action flags pending server acknowledgment
- Inputs: (none)
- Outputs/Return: Count of unconfirmed action flags (write_tick - read_tick)
- Side effects: (none)
- Calls: `spoke_get_unconfirmed_flags_queue()`
- Notes: Used to detect when client is ahead of server confirmation

### StarGameProtocol::PeekUnconfirmedActionFlag
- Signature: `uint32 PeekUnconfirmedActionFlag(int32 offset)`
- Purpose: Inspect an unconfirmed action flag without consuming it
- Inputs: `offset` ΓÇö index relative to read tick
- Outputs/Return: Action flags at (read_tick + offset)
- Side effects: (none)
- Calls: `spoke_get_unconfirmed_flags_queue()`, queue's `peek()` method
- Notes: Used for predictive client-side simulation

### StarGameProtocol::UpdateUnconfirmedActionFlags
- Signature: `void UpdateUnconfirmedActionFlags()`
- Purpose: Dequeue unconfirmed flags that have been confirmed by server
- Inputs: (none)
- Outputs/Return: (none)
- Side effects: Advances read tick on unconfirmed queue to smallest confirmed tick
- Calls: `spoke_get_unconfirmed_flags_queue()`, `spoke_get_smallest_unconfirmed_tick()`
- Notes: Processes server confirmations; safe if queue is empty or no new confirmations

### make_player_really_net_dead (module-level)
- Signature: `void make_player_really_net_dead(size_t inPlayerIndex)`
- Purpose: Mark a player as permanently disconnected in network topology
- Inputs: `inPlayerIndex` ΓÇö index in sTopology->players array
- Outputs/Return: (none)
- Side effects: Sets `sTopology->players[inPlayerIndex].net_dead = true`
- Calls: (none)
- Notes: Called when spoke detects hub or peer disconnection; no queue cleanup (separate concern)

### call_distribution_response_function_if_available (module-level)
- Signature: `void call_distribution_response_function_if_available(byte* inBuffer, uint16 inBufferSize, int16 inDistributionType, uint8 inSendingPlayerIndex)`
- Purpose: Invoke registered callback for received distribution message
- Inputs:
  - `inBuffer` ΓÇö message payload
  - `inBufferSize` ΓÇö payload size
  - `inDistributionType` ΓÇö message type
  - `inSendingPlayerIndex` ΓÇö originating player
- Outputs/Return: (none)
- Side effects: Calls distribution callback if type is registered
- Calls: `NetGetDistributionInfoForType()`, `NetDistributionInfo::distribution_proc`
- Notes: No-op if type not registered; assumes callback handles per-type logic

### StarGameProtocol::GetParser (module-level)
- Signature: `static XML_ElementParser* GetParser()`
- Purpose: Get XML element parser for star protocol configuration
- Inputs: (none)
- Outputs/Return: Pointer to static `sStarParser`
- Side effects: Adds hub and spoke child parsers on first call (lazy init)
- Calls: `Hub_GetParser()`, `Spoke_GetParser()`, `sStarParser.AddChild()`
- Notes: Thread-unsafe; assumes single-threaded initialization

### DefaultStarPreferences / WriteStarPreferences (module-level)
- Signatures:
  - `void DefaultStarPreferences()`
  - `void WriteStarPreferences(FILE* F)`
- Purpose: Initialize and serialize network protocol preferences
- Inputs: (for WriteStarPreferences) `F` ΓÇö output file stream
- Outputs/Return: (none, side effects on preferences system / file)
- Side effects: Delegates to hub and spoke preference handlers
- Calls: `DefaultHubPreferences()`, `DefaultSpokePreferences()`, `WriteHubPreferences()`, `WriteSpokePreferences()`
- Notes: Trivial wrappers; actual logic in hub/spoke modules

## Control Flow Notes
- **Initialization**: `Enter()` ΓåÆ `Sync()` branches into hub and/or spoke initialization depending on role
- **Runtime**: `PacketHandler()` receives packets each frame; `DistributeInformation()` sends updates; action queue adapters accumulate player input
- **Action management**: `GetUnconfirmedActionFlagsCount()`, `PeekUnconfirmedActionFlag()`, `UpdateUnconfirmedActionFlags()` manage client-side prediction and server confirmation
- **Shutdown**: `UnSync()` cleans hub/spoke and deallocates queues; `Exit1()`/`Exit2()` perform final teardown
- Not inferable: exact frame timing, when PacketHandler is called, interaction with game state machine

## External Dependencies
- **Network**: `network_star.h` ΓÇö hub/spoke implementations (`hub_initialize`, `spoke_initialize`, `hub_received_network_packet`, `spoke_received_network_packet`, etc.)
- **Queue abstraction**: `TickBasedCircularQueue.h` ΓÇö `WritableTickBasedCircularQueue`, `ConcreteTickBasedCircularQueue`
- **Player / Actions**: `player.h` ΓÇö `GetRealActionQueues()` (returns active action queue set)
- **Interface**: `interface.h` ΓÇö `process_action_flags()` (delivers action flags to player update logic)
- **XML**: Not explicitly included; `XML_ElementParser` assumed defined elsewhere
- **Networking**: `cseries.h` (platform types, build config); `DDPPacketBuffer` type (assumed from headers not shown)
- **Distribution**: `NetGetDistributionInfoForType()`, `NetDistributionInfo` (defined elsewhere)
