# Source_Files/Network/network_star_hub.cpp

## File Purpose
Implements the hub-side (server/relay) of Aleph One's star-topology network protocol. The hub collects action flags from all connected players, detects disconnections via timeout, broadcasts synchronized game state back to each player with acknowledgment tracking, and manages timing adjustments and lossy byte stream distribution.

## Core Responsibilities
- **Hub lifecycle**: Initialize/cleanup hub state, manage timer task, coordinate graceful shutdown
- **Packet reception**: Parse incoming spoke packets, extract action flags, process identification and acknowledgments
- **Player state tracking**: Monitor connection status, detect net-dead players, track smallest unacknowledged tick per player
- **Flag aggregation**: Maintain tick-based queues of action flags; track which players have provided data for each tick
- **Periodic broadcasting**: On each tick (~33ms), send game data packets to all connected spokes with:
  - Aggregated action flags from other players
  - Timing adjustment messages
  - Net-dead player announcements
  - Lossy byte stream data
- **Timing & latency**: Calculate per-player latency and jitter; manage windowed latency data; trigger timing adjustments
- **Bandwidth optimization**: Support "bandwidth reduction" mode to send fewer/larger updates instead of incremental ones
- **Lossy distribution**: Buffer and forward lossy byte stream messages (e.g., voice chat) to destination spokes

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `HubPreferences` | struct | Configuration for pregame/ingame window sizes, timeouts, send periods, nth-element selection |
| `NetworkPlayer_hub` | struct | Per-player state: address, connection status, unacknowledged tick, timing adjustment pending, latency buffer, network stats |
| `HubLossyByteStreamChunkDescriptor` | struct | Metadata for a chunk of lossy byte stream data: length, type, destinations bitmask, sender |
| `AddressToPlayerIndexType` | typedef | `std::map<NetAddrBlock, int>` mapping network address to player index |
| `TickBasedActionQueueCollection` | typedef | `std::vector<TickBasedActionQueue>` for per-player action flag queues |
| `MutableElementsTickBasedCircularQueue<uint32>` | class | Circular queue with mutable elements; tracks which players have acked each tick (as bitmask) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sNetworkTicker` | int32 | static | Monotonic counter; increments every hub_tick regardless of game clock |
| `sLastNetworkTickSent` | int32 | static | sNetworkTicker value when we last sent packets; used for recovery sends |
| `sSmallestRealGameTick` | int32 | static | First tick of actual game (after kPregameTicks synchronization period) |
| `sSmallestPostGameTick` | int32 | static | Marker tick at game end; hub cleanups when all players ack past this |
| `sFlagsQueues` | TickBasedActionQueueCollection | static | Per-player circular queues of action_flags_t, keyed by tick |
| `sLateFlagsQueues` | TickBasedActionQueueCollection | static | Queues for late-arriving flags from lagging players |
| `sFlagSendTimeQueue` | ConcreteTickBasedCircularQueue<int32> | static | Tracks sNetworkTicker value when each tick's flags first sent |
| `sPlayerDataDisposition` | MutableElementsTickBasedCircularQueue<uint32> | static | Per-tick bitmask: 1 bit per player; cleared as they ack; tick complete when all 0 |
| `sSmallestIncompleteTick` | int32 | static | First tick not yet acked by all connected players |
| `sSmallestUnsentTick` | int32 | static | Used with mSendPeriod to throttle packet sends |
| `sConnectedPlayersBitmask` | uint32 | static | Bitmask of connected players (1 bit per index) |
| `sLaggingPlayersBitmask` | uint32 | static | Bitmask of lagging players (for bandwidth reduction mode) |
| `sNetworkPlayers` | NetworkPlayerCollection | static | Array of per-player state (`NetworkPlayer_hub` instances) |
| `sAddressToPlayerIndex` | AddressToPlayerIndexType | static | Map from UDP address to player index (built up as spokes identify) |
| `sLocalPlayerIndex` | size_t | static | Index of player on hub's local machine (or NONE for standalone hub) |
| `sReferencePlayerIndex` | size_t | static | Player index used for timing reference calculations |
| `sOutgoingFrame` | DDPFramePtr | static | Reusable frame buffer for outgoing UDP packets |
| `sLocalOutgoingBuffer` | DDPPacketBuffer | static (non-standalone) | Packet buffer for local spoke communication (bypasses UDP) |
| `sNeedToSendLocalOutgoingBuffer` | bool | static (non-standalone) | Flag to trigger local spoke packet delivery |
| `sOutgoingLossyByteStreamData` | CircularByteBuffer | static | Ring buffer holding lossy byte stream payload data |
| `sOutgoingLossyByteStreamDescriptors` | CircularQueue<...> | static | Circular queue of descriptors (one per lossy chunk in data buffer) |
| `sScratchBuffer` | byte[kLossyByteStreamDataBufferSize] | static | Temporary buffer for copying lossy data (serialization/deserialization) |
| `sHubActive` | bool | static | Gate for packet processing; set false to stop accepting new packets |
| `sHubInitialized` | bool | static | Flag indicating hub_initialize has been called |
| `sHubTickTask` | myTMTaskPtr | static | Handle to recurring timer task (calls hub_tick every ~33ms) |
| `sHubPreferences` | HubPreferences | static | Configuration loaded from XML (window sizes, timeouts, send periods, etc.) |
| `sLastRealUpdate` | int32 | static | sNetworkTicker value at last tick with real (non-synthetic) data; used for bandwidth reduction |
| `sLastFlagsReceived` | vector<action_flags_t> | static | Last actual flags received from each player (for synthetic flag generation) |
| `sPlayerReflectedFlags` | MutableElementsTickBasedCircularQueue<uint32> | static | Per-tick bitmask of players whose flags need to be reflected back to them |

## Key Functions / Methods

### hub_initialize
- **Signature:** `void hub_initialize(int32 inStartingTick, size_t inNumPlayers, const NetAddrBlock* const* inPlayerAddresses, size_t inLocalPlayerIndex)`
- **Purpose:** Set up all hub state and begin receiving/broadcasting.
- **Inputs:** Starting game tick, number of players, array of initial addresses (may be NULL), local player index.
- **Outputs/Return:** None.
- **Side effects:** Allocates queues, initializes all global arrays and maps, creates timer task, sets sHubActive=true.
- **Calls:** `obj_clear()`, `NetDDPNewFrame()`, `myXTMSetup()` (timer setup).
- **Notes:** Must be called once before hub_tick or hub_received_network_packet. Spokes may not yet have sent identification packets, so many addresses start unknown.

### hub_cleanup
- **Signature:** `void hub_cleanup(bool inGraceful, int32 inSmallestPostGameTick)`
- **Purpose:** Shut down the hub, optionally waiting for acknowledgments.
- **Inputs:** `inGraceful` (true = wait for ACKs before exit; false = stop immediately); `inSmallestPostGameTick` (game end tick, for graceful path).
- **Outputs/Return:** None.
- **Side effects:** If graceful, polls `hub_check_for_completion()` until all connected players ack past the end tick or become disconnected. Sets `sHubActive=false`, removes timer task, clears queues and players.
- **Calls:** `take_mytm_mutex()`, `hub_check_for_completion()`, `release_mytm_mutex()`, `myTMRemove()`, `myTMCleanup()`, `NetDDPDisposeFrame()`.
- **Notes:** Graceful mode uses spin-loop with SDL_Delay; non-graceful immediately stops packet processing.

### hub_received_network_packet
- **Signature:** `void hub_received_network_packet(DDPPacketBufferPtr inPacket)`
- **Purpose:** Process an incoming UDP packet from a spoke.
- **Inputs:** Pointer to received datagram (contains source address).
- **Outputs/Return:** None.
- **Side effects:** Parses packet, updates player state, enqueues flags, triggers local spoke delivery if needed; raises flag for packet error if CRC fails.
- **Calls:** `AIStreamBE` (big-endian parser), `calculate_data_crc_ccitt()`, `hub_received_identification_packet()`, `hub_received_game_data_packet_v1()`, `check_send_packet_to_spoke()`.
- **Notes:** Dispatches on packet magic (ID, V1 game data). Silently discards malformed/bad-CRC packets. No reentrancy with hub_tick (protected by mytm_mutex policy).

### hub_received_identification_packet
- **Signature:** `void hub_received_identification_packet(AIStream& ps, NetAddrBlock address)`
- **Purpose:** Record the network address of a spoke once it identifies itself.
- **Inputs:** Stream with player index; source address from UDP datagram.
- **Outputs/Return:** None.
- **Side effects:** Updates `sAddressToPlayerIndex` map and `sNetworkPlayers[index].mAddress`, marks address as known.
- **Calls:** (none, simple assignment).
- **Notes:** Called during hub_received_network_packet. Handles NAT by allowing spokes to tell us their true address.

### hub_received_game_data_packet_v1
- **Signature:** `void hub_received_game_data_packet_v1(AIStream& ps, int inSenderIndex)`
- **Purpose:** Parse action flags and acknowledgments from a spoke's game data packet.
- **Inputs:** Big-endian stream; sender's player index.
- **Outputs/Return:** None.
- **Side effects:** Calls `player_acknowledged_up_to_tick()` for ACK, then `process_messages()` for in-band messages, then `player_provided_flags_from_tick_to_tick()` for action flags. Updates latency buffers.
- **Calls:** `player_acknowledged_up_to_tick()`, `process_messages()`, `player_provided_flags_from_tick_to_tick()`.
- **Notes:** ACK is mandatory; messages and flags are optional. Rejects ACK if beyond sSmallestIncompleteTick (safety check).

### player_acknowledged_up_to_tick
- **Signature:** `void player_acknowledged_up_to_tick(size_t inPlayerIndex, int32 inSmallestUnacknowledgedTick)`
- **Purpose:** Mark all ticks up to the given one as acked by a player.
- **Inputs:** Player index, smallest tick they haven't yet acked.
- **Outputs/Return:** None.
- **Side effects:** Updates `sNetworkPlayers[inPlayerIndex].mSmallestUnacknowledgedTick`. Advances `sPlayerDataDisposition.getReadIndex()` as ticks become "complete" (all players have acked). May call `hub_check_for_completion()` if game is shutting down.
- **Calls:** (none directly visible, but modifies queue state).
- **Notes:** Bitmask in `sPlayerDataDisposition` is decremented as each player acks; when it reaches 0, the tick is complete.

### player_provided_flags_from_tick_to_tick
- **Signature:** `bool player_provided_flags_from_tick_to_tick(size_t inPlayerIndex, int32 inFirstNewTick, int32 inSmallestUnreceivedTick)`
- **Purpose:** Enqueue action flags from a range of ticks; update bitmask and check for tick completion.
- **Inputs:** Player index, first new tick, smallest tick not yet received.
- **Outputs/Return:** `bool` indicating whether to trigger a send.
- **Side effects:** Enqueues flags into `sFlagsQueues[inPlayerIndex]`. Updates `sPlayerDataDisposition` bitmask, potentially advancing `sSmallestIncompleteTick`. Handles late flags in `sLateFlagsQueues`. May update `sSmallestUnheardTick` and latency measurements.
- **Calls:** `getFlagsQueue()`, `getLateFlagsQueue()`, `std::min()`.
- **Notes:** Tracks `mNthElementFinder` for latency; handles "lagging" players separately.

### make_player_netdead
- **Signature:** `void make_player_netdead(int inPlayerIndex)`
- **Purpose:** Mark a player as disconnected and synthesize missing acknowledgments/flags to complete pending ticks.
- **Inputs:** Player index.
- **Outputs/Return:** None.
- **Side effects:** Sets `mConnected=false`, clears from `sConnectedPlayersBitmask`, removes from `sAddressToPlayerIndex`, sets `mNetDeadTick`. Calls `player_provided_flags_from_tick_to_tick()` to synthesize ACKs and flags so pending ticks can complete.
- **Calls:** `MyTMMutexTaker` (RAII lock), `player_provided_flags_from_tick_to_tick()`, `player_acknowledged_up_to_tick()`.
- **Notes:** Triggered by timeout in hub_tick or by queue overflow. Allows game to continue despite one player's departure.

### hub_tick
- **Signature:** `bool hub_tick()`
- **Purpose:** Periodic hub update (~30 ticks/second). Check for disconnections, possibly send synthetic flags, send packets, update latency stats.
- **Inputs:** None.
- **Outputs/Return:** `bool` (always true, indicating timer should continue).
- **Side effects:** Increments `sNetworkTicker`. Detects net-dead players (by silence or queue overflow); calls `make_player_netdead()`. If bandwidth reduction mode, synthesizes flags for lagging players. Calls `send_packets()`. Updates `mLatencyBuffer` and calculates jitter/ping stats.
- **Calls:** `make_player_netdead()`, `make_up_flags_for_first_incomplete_tick()`, `send_packets()`, `check_send_packet_to_spoke()`, `std::min()`, `std::max()`, `std::sqrt()`, `accumulate()`.
- **Notes:** Only entry point called by timer; all state updates happen here and in async packet handler. Uses latency data to adapt recovery send period in bandwidth reduction mode.

### send_packets
- **Signature:** `void send_packets()`
- **Purpose:** Construct and send one outgoing game data packet to each connected spoke.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** For each player, builds a packet with: ACK of their flags, timing adjustments, net-dead messages, lossy byte stream data, action flags from all other players (or subset if bandwidth reduction). Sends via UDP (or local spoke interface). Updates `sLastNetworkTickSent`, `sSmallestUnsentTick`. Dequeues one lossy descriptor if present.
- **Calls:** `AOStreamBE` (big-endian writer), `getFlagsQueue()`, `send_frame_to_local_spoke()`, `NetDDPSendFrame()`, `calculate_data_crc_ccitt()`.
- **Notes:** Handles both incremental and recovery (full) updates. Reflects flags back if requested. May skip sending own flags to self unless reflection needed. Encodes tick-major order (easy to decode on spoke side).

### process_messages
- **Signature:** `void process_messages(AIStream& ps, int inSenderIndex)`
- **Purpose:** Parse a stream of variable-length messages until `kEndOfMessagesMessageType`.
- **Inputs:** Input stream, sender player index.
- **Outputs/Return:** None.
- **Side effects:** Reads message type codes from stream; dispatches to `process_optional_message()`.
- **Calls:** `process_optional_message()`.
- **Notes:** Trivial loop; real logic in process_optional_message.

### process_optional_message
- **Signature:** `void process_optional_message(AIStream& ps, int inSenderIndex, uint16 inMessageType)`
- **Purpose:** Dispatch a single optional message by type.
- **Inputs:** Stream, sender index, message type code.
- **Outputs/Return:** None.
- **Side effects:** Reads message length, then dispatches to handler (currently only lossy byte stream messages) or skips unknown types.
- **Calls:** `process_lossy_byte_stream_message()`.
- **Notes:** Extension point for new message types.

### process_lossy_byte_stream_message
- **Signature:** `void process_lossy_byte_stream_message(AIStream& ps, int inSenderIndex, uint16 inLength)`
- **Purpose:** Receive a chunk of lossy byte stream data (e.g., voice) and queue it for distribution.
- **Inputs:** Stream, sender index, message payload length.
- **Outputs/Return:** None.
- **Side effects:** Parses descriptor (type, destinations bitmask), checks buffer space, enqueues into `sOutgoingLossyByteStreamData` and descriptor queue (or discards if no space).
- **Calls:** `ps.tellg()`, `ps.read()`, `ps.ignore()`, logging functions.
- **Notes:** Silently discards if buffers full (lossy semantics). Reads payload into scratch buffer, then copies to circular buffer.

### hub_check_for_completion
- **Signature:** `void hub_check_for_completion()`
- **Purpose:** During graceful shutdown, check if all connected players have acked up to `sSmallestPostGameTick`.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** If condition met, sets `sHubActive=false` (signaling cleanup to proceed).
- **Calls:** (none).
- **Notes:** Called from hub_cleanup and potentially from player_acknowledged_up_to_tick.

### Hub_GetParser
- **Signature:** `XML_ElementParser* Hub_GetParser()`
- **Purpose:** Return parser for hub configuration in XML.
- **Inputs:** None.
- **Outputs/Return:** Pointer to static `XML_HubConfigurationParser` instance.
- **Calls:** (none).

### WriteHubPreferences
- **Signature:** `void WriteHubPreferences(FILE* F)`
- **Purpose:** Serialize current hub preferences to XML.
- **Inputs:** Output file pointer.
- **Outputs/Return:** None.
- **Side effects:** Writes `<hub .../>` element with non-default attributes, plus comment with all defaults.
- **Calls:** `fprintf()`.

### DefaultHubPreferences
- **Signature:** `void DefaultHubPreferences()`
- **Purpose:** Reset all hub preferences to hardcoded defaults.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Overwrites all entries in `sHubPreferences` from `sDefaultHubPreferences` array.
- **Calls:** (none).

**Inline helpers / trivial functions:**
- `getNetworkPlayer(size_t)`: Returns reference to `sNetworkPlayers[index]` (bounds-checked).
- `getFlagsQueue(size_t)`: Returns reference to `sFlagsQueues[index]`.
- `getLateFlagsQueue(size_t)`: Returns reference to `sLateFlagsQueues[index]`.
- `operator<(const NetAddrBlock&, const NetAddrBlock&)`: Memcmp-based comparison.
- `send_frame_to_local_spoke(...)`: Queues a frame for the local spoke (non-standalone only).
- `check_send_packet_to_spoke()`: Delivers queued local packet (non-standalone only).
- `hub_get_minimum_send_period()`, `hub_set_minimum_send_period()`: Accessors for config.
- `hub_stats(int)`: Returns `const NetworkStats&` for a player.
- `add_squares(int, int)`: Helper for jitter calculation (accumulate adapter).

## Control Flow Notes

The hub operates in a **tick-based event-driven model**:

1. **Initialization** (`hub_initialize`): Called once at game start. Sets up timer to call `hub_tick` at ~30 Hz. Sets `sHubActive=true` to accept packets.

2. **Concurrent threads**:
   - **Packet receiver** (async, external): Calls `hub_received_network_packet()` when UDP arrives. Protected by `mytm_mutex` from interfering with hub_tick.
   - **Hub ticker** (timer callback): Calls `hub_tick()` every ~33ms. Protected by `mytm_mutex` from interfering with packet handler.

3. **Per-tick flow** (in `hub_tick`):
   - Increment `sNetworkTicker`.
   - Detect silent/lagging players; call `make_player_netdead()` if timeout exceeded.
   - If bandwidth reduction enabled and majority of players ready, synthesize flags via `make_up_flags_for_first_incomplete_tick()`.
   - Call `send_packets()` to broadcast.
   - Update latency/jitter stats (every `kJitterUpdateInterval` ticks).

4. **Packet reception flow** (in `hub_received_network_packet`):
   - Parse packet magic and CRC; discard if bad.
   - Dispatch by magic: identification, or V1 game data.
   - For game data: extract ACK ΓåÆ call `player_acknowledged_up_to_tick()` ΓåÆ parse messages ΓåÆ parse flags ΓåÆ call `player_provided_flags_from_tick_to_tick()`.
   - Enqueue action flags into per-player tick-based queues.
   - Update bitmask in `sPlayerDataDisposition`; advance read index as ticks complete (all players acked).

5. **Shutdown** (`hub_cleanup`):
   - If graceful: set `sSmallestPostGameTick` and spin until all players ack or disconnect.
   - Stop the timer task and wait for it to finish.
   - Clear all state.

**Key synchronization invariant**: Ticks advance through `sPlayerDataDisposition` only when all connected players have both provided data and acknowledged. This ensures every spoke receives the same set of action flags for each tick.

## External Dependencies

- **Network protocol** (network_star.h, network_private.h): Packet magic constants, action flag types, protocol definitions.
- **Queue data structures** (TickBasedCircularQueue.h): `ConcreteTickBasedCircularQueue<T>`, `MutableElementsTickBasedCircularQueue<uint32>`.
- **Serialization** (AStream.h): `AIStreamBE`, `AOStreamBE` for big-endian read/write.
- **Logging** (Logging.h): `logContextNMT()`, `logWarningNMT*()`, `logDumpNMT*()`, etc. (non-main-thread variants).
- **Timing** (mytm.h): `myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`, `MyTMMutexTaker` (RAII mutex).
- **Latency analysis** (WindowedNthElementFinder.h): Used in `NetworkPlayer_hub.mNthElementFinder`.
- **Lossy buffering** (CircularByteBuffer.h): `CircularByteBuffer` for payload storage.
- **XML configuration** (XML_ElementParser.h): `XML_ElementParser` base class.
- **Network I/O** (sdl_network.h): `DDPFramePtr`, `DDPPacketBuffer`, `NetDDPNewFrame()`, `NetDDPDisposeFrame()`, `NetDDPSendFrame()`.
- **CRC** (crc.h): `calculate_data_crc_ccitt()` for packet integrity.
- **Action flags** (player.h): Definitions of action flag masks.
- **Timer utilities** (SDL_timer.h): `SDL_Delay()` for sleep in shutdown polling.
- **Standard C++ library**: `<vector>`, `<map>`, `<algorithm>`, `<deque>`, `<numeric>`, `<cmath>`.

**Defined elsewhere (external symbols used)**:
- `spoke_received_network_packet()`: Deliver packet to local spoke on hub machine.
- Timer system (mytm): Provides periodic callback infrastructure.
- Network stack (NetDDP*): UDP send/receive backend.
