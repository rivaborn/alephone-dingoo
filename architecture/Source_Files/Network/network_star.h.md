# Source_Files/Network/network_star.h

## File Purpose
Defines the star network topology interface for Aleph One's multiplayer synchronization. Declares functions and types for hub (server) and spoke (client) roles, action flag queuing, packet handling, and network timing in a star topology where one hub coordinates all communication.

## Core Responsibilities
- Define message type constants for hubΓÇôspoke communication (end-of-messages, timing adjustment, player disconnect, lossy streams, game data packets)
- Declare initialization/cleanup for hub and spoke roles
- Declare packet reception handlers for both hub and spoke
- Define tick-based action flag queue types for reliable action synchronization
- Provide latency measurement and unconfirmed action tracking
- Support lossy streaming channels for non-critical data (e.g., voice/video)
- Provide XML configuration parsers and preferences I/O for hub and spoke

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `action_flags_t` | typedef | 32-bit action flags for a player in a game tick |
| `TickBasedActionQueue` | typedef | Read-only tick-keyed queue of action flags |
| `WritableTickBasedActionQueue` | typedef | Write-capable tick-keyed queue of action flags |

## Global / File-Static State
None.

## Key Functions / Methods

### hub_initialize
- Signature: `void hub_initialize(int32 inStartingTick, size_t inNumPlayers, const NetAddrBlock* const* inPlayerAddresses, size_t inLocalPlayerIndex)`
- Purpose: Initialize the hub (server) with player addresses and starting tick.
- Inputs: Starting game tick, number of players, array of player network addresses, local player index.
- Outputs/Return: None.
- Side effects: Sets up hub state, action queues, and network listeners.
- Calls: Not inferable from this file.
- Notes: Called once at network game start on the hub machine.

### hub_cleanup
- Signature: `void hub_cleanup(bool inGraceful, int32 inSmallestPostGameTick)`
- Purpose: Shut down the hub, optionally sending final state to clients.
- Inputs: Graceful flag (true = send farewells), smallest confirmed tick post-game.
- Outputs/Return: None.
- Side effects: Closes network listeners, flushes queues.
- Calls: Not inferable from this file.

### hub_received_network_packet
- Signature: `void hub_received_network_packet(DDPPacketBufferPtr inPacket)`
- Purpose: Handle an incoming network packet at the hub.
- Inputs: Packet buffer containing source address and datagram.
- Outputs/Return: None.
- Side effects: Updates action queues, broadcasts to spokes, detects disconnects.
- Calls: Not inferable from this file.

### spoke_initialize
- Signature: `void spoke_initialize(const NetAddrBlock& inHubAddress, int32 inFirstTick, size_t inNumberOfPlayers, WritableTickBasedActionQueue* const inPlayerQueues[], bool inPlayerConnectedStatus[], size_t inLocalPlayerIndex, bool inHubIsLocal)`
- Purpose: Initialize the spoke (client) to connect to a hub.
- Inputs: Hub address, first game tick, player count, array of action queues (one per player), connection status array, local player index, whether hub is on same machine.
- Outputs/Return: None.
- Side effects: Opens connection to hub, synchronizes timing over `kPregameTicks`.
- Calls: Not inferable from this file.

### spoke_received_network_packet
- Signature: `void spoke_received_network_packet(DDPPacketBufferPtr inPacket)`
- Purpose: Handle an incoming network packet at a spoke.
- Inputs: Packet buffer from hub.
- Outputs/Return: None.
- Side effects: Updates remote player action queues, applies timing adjustments.
- Calls: Not inferable from this file.

### spoke_get_net_time
- Signature: `int32 spoke_get_net_time()`
- Purpose: Retrieve the current network-synchronized game tick on this spoke.
- Inputs: None.
- Outputs/Return: Game tick (int32).
- Side effects: None.
- Calls: Not inferable from this file.

### spoke_distribute_lossy_streaming_bytes_to_everyone
- Signature: `void spoke_distribute_lossy_streaming_bytes_to_everyone(int16 inDistributionType, byte* inBytes, uint16 inLength, bool inExcludeLocalPlayer, bool onlySendToTeam)`
- Purpose: Send lossy data (e.g., voice) to all remote players via the hub.
- Inputs: Distribution type ID, byte buffer, length, exclude-local flag, team-only flag.
- Outputs/Return: None.
- Side effects: Queues data for transmission to hub; no guarantee of delivery.
- Calls: Not inferable from this file.

### spoke_distribute_lossy_streaming_bytes
- Signature: `void spoke_distribute_lossy_streaming_bytes(int16 inDistributionType, uint32 inDestinationsBitmask, byte* inBytes, uint16 inLength)`
- Purpose: Send lossy data to a specific set of players via bitmask.
- Inputs: Distribution type, destinations bitmask, byte buffer, length.
- Outputs/Return: None.
- Side effects: Queues data for transmission; allows fine-grained recipient control.
- Calls: Not inferable from this file.

### spoke_latency
- Signature: `int32 spoke_latency()`
- Purpose: Measure round-trip latency from spoke to hub in milliseconds.
- Inputs: None.
- Outputs/Return: Latency (ms); `kNetLatencyInvalid` if not yet measured.
- Side effects: None.
- Calls: Not inferable from this file.

### hub_latency
- Signature: `int32 hub_latency(int player_index)`
- Purpose: Measure latency from hub to a specific player.
- Inputs: Player index.
- Outputs/Return: Latency (ms); `kNetLatencyInvalid` if not valid, `kNetLatencyDisconnected` if player disconnected.
- Side effects: None.
- Calls: Not inferable from this file.

### spoke_get_unconfirmed_flags_queue
- Signature: `TickBasedActionQueue* spoke_get_unconfirmed_flags_queue()`
- Purpose: Return a read-only queue of action flags that have not yet been confirmed by the hub.
- Inputs: None.
- Outputs/Return: Pointer to tick-keyed action queue.
- Side effects: None.
- Calls: Not inferable from this file.
- Notes: Used by prediction systems to extrapolate player actions ahead of confirmation.

### spoke_get_smallest_unconfirmed_tick
- Signature: `int32 spoke_get_smallest_unconfirmed_tick()`
- Purpose: Get the oldest game tick whose action flags have not been confirmed.
- Inputs: None.
- Outputs/Return: Game tick (int32).
- Side effects: None.
- Calls: Not inferable from this file.

## Control Flow Notes
**Initialization phase:**  
1. Hub calls `hub_initialize()` once to listen for spokes.
2. Each spoke calls `spoke_initialize()` once to connect and resynchronize over `kPregameTicks` (3 seconds at 30 ticks/sec).

**Per-frame packet handling:**  
- Hub receives packets via `hub_received_network_packet()`, distributes action flags to all spokes' queues, and broadcasts updated state.
- Spoke receives packets via `spoke_received_network_packet()`, updates remote player action queues from hub, applies timing corrections.

**Lossy streaming:**  
- Spokes can send voice/video via `spoke_distribute_lossy_streaming_bytes_to_everyone()` or the lower-level bitmask variant; hub distributes lossy packets to other spokes.

**Shutdown:**  
- Hub calls `hub_cleanup()` (optionally graceful); spoke calls `spoke_cleanup()`.

## External Dependencies
- **TickBasedCircularQueue.h**: `ConcreteTickBasedCircularQueue<T>` and `WritableTickBasedCircularQueue<T>` templates for tick-indexed queues.
- **ActionQueues.h**: Additional queue utilities (included but action_flags_t uses the TickBasedCircularQueue).
- **sdl_network.h**: `DDPPacketBufferPtr`, `NetAddrBlock`, packet I/O definitions.
- **map.h**: `TICKS_PER_SECOND` constant (30 ticks/sec in standalone hub mode).
- **XML_ElementParser** (forward-declared): Used for configuration parsing.
- Defined elsewhere: `kNetLatencyInvalid`, `kNetLatencyDisconnected` (constants), packet magic constants.
