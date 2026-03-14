# Source_Files/Network/network_star_spoke.cpp

## File Purpose
Implements the spoke (client) node of a star-topology multiplayer network protocol for the Aleph One game engine. Each player runs this code to maintain a connection to a central hub, exchange action flags (input), and distribute streaming data.

## Core Responsibilities
- Initialize and maintain spoke connection state to a single hub, including local or remote hub support
- Receive and validate game data packets from hub (CRC verification)
- Manage local player action flag queues (outgoing, unconfirmed, confirmed)
- Receive remote player action flags from hub and enqueue to per-player queues
- Detect hub disconnection via timeout and initiate graceful disconnect with net-dead flags
- Send periodic identification packets (pre-connection) and game data packets (post-connection)
- Maintain per-player net-dead status and synthetic flags for disconnected players
- Buffer and forward lossy (unreliable) byte-stream data to specified player destinations
- Measure and adjust network timing via windowed nth-element latency filter
- Handle XML configuration for net-death timeouts, send periods, and timing windows

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SpokePreferences` | struct | Configuration: pregame/in-game net-death timeouts, recovery send period, timing window/nth-element |
| `NetworkPlayer_spoke` | struct | Per-player state: zombie flag, connected status, net-dead tick, action queue pointer |
| `IncomingGameDataPacketProcessingContext` | struct | Transient context for message processing: end-of-messages flag, timing-adjustment-received flag |
| `SpokeLossyByteStreamChunkDescriptor` | struct | Metadata for each lossy byte-stream chunk: length, type, destination bitmask |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sSpokePreferences` | `SpokePreferences` | static | Configuration (timeouts, periods, timing parameters) |
| `sOutgoingFlags` | `TickBasedActionQueue` | static | Action flags queued for transmission to hub |
| `sUnconfirmedFlags` | `TickBasedActionQueue` | static | Local player flags awaiting hub echo-back confirmation |
| `sLocallyGeneratedFlags` | `DuplicatingTickBasedCircularQueue<action_flags_t>` | static | Meta-queue combining outgoing + unconfirmed (used during tick) |
| `sNetworkPlayers` | `vector<NetworkPlayer_spoke>` | static | Per-player state (connected, net-dead tick, queue pointers) |
| `sNetworkTicker` | `int32` | static | Local game tick counter incremented by timer |
| `sLastNetworkTickHeard` | `int32` | static | Tick when last packet arrived from hub (timeout detection) |
| `sLastNetworkTickSent` | `int32` | static | Tick when last packet sent (recovery send period) |
| `sConnected` | `bool` | static | True if hub presumed alive; false after timeout |
| `sSpokeActive` | `bool` | static | True during operation; false after cleanup |
| `sHubAddress` | `NetAddrBlock` | static | Hub's network address (updated by identification packet source) |
| `sHubIsLocal` | `bool` | static | True if hub is in-process; false if remote |
| `sSmallestUnreceivedTick` | `int32` | static | Lowest tick for which we have enqueued flags to player queues |
| `sSmallestUnconfirmedTick` | `int32` | static | Lowest tick in unconfirmed queue not yet echoed back by hub |
| `sOutgoingLossyByteStreamData` | `CircularByteBuffer` | static | Ring buffer holding lossy streaming chunk payloads |
| `sOutgoingLossyByteStreamDescriptors` | `CircularQueue<...>` | static | Ring queue of metadata for lossy chunks |
| `sRequestedTimingAdjustment` | `int8` | static | Timing adjustment sent in current outgoing packet |
| `sOutstandingTimingAdjustment` | `int8` | static | Remaining timing adjustment to apply locally |
| `sNthElementFinder` | `WindowedNthElementFinder<int32>` | static | Latency measurement filter (nth-largest in sliding window) |
| `sTimingMeasurementValid` | `bool` | static | True when latency window full and measurement valid |
| `sTimingMeasurement` | `int32` | static | Computed latency (in ticks) used to adjust net time |
| `sHeardFromHub` | `bool` | static | True after first packet from hub; controls ID-packet phase |
| `sDisplayLatencyBuffer` | `vector<int32>` | static | Rolling 1-second window of latency samples for UI display |
| `sMessageTypeToMessageHandler` | `map<uint16, StarMessageHandler>` | static | Dispatch table for incoming message types |

## Key Functions / Methods

### spoke_initialize
- **Signature:** `void spoke_initialize(const NetAddrBlock& inHubAddress, int32 inFirstTick, size_t inNumberOfPlayers, WritableTickBasedActionQueue* const inPlayerQueues[], bool inPlayerConnected[], size_t inLocalPlayerIndex, bool inHubIsLocal)`
- **Purpose:** Initialize spoke state before game starts; allocate queues, register message handlers, start timer.
- **Inputs:** Hub address, starting tick, player count, per-player queues, connected flags, local player index, local-hub flag
- **Outputs/Return:** None (void)
- **Side effects:** Initializes all static state; registers 4 message handlers; starts periodic timer task
- **Calls:** `NetDDPNewFrame()`, `myXTMSetup(1000/TICKS_PER_SECOND, spoke_tick)`, logging
- **Notes:** Pregame ticks calculated as `inFirstTick - kPregameTicks`. Asserts local player queue non-NULL and connected.

### spoke_cleanup
- **Signature:** `void spoke_cleanup(bool inGraceful)`
- **Purpose:** Shut down spoke; stop timer, send final packet, clear state.
- **Inputs:** `inGraceful` (unused)
- **Outputs/Return:** None (void)
- **Side effects:** Stops timer task, clears all static collections, sets `sSpokeActive = false`
- **Calls:** `take_mytm_mutex()`, `myTMRemove()`, `send_packet()`, `check_send_packet_to_hub()`, `release_mytm_mutex()`, `myTMCleanup()`, `NetDDPDisposeFrame()`
- **Notes:** Sends one final packet to prevent hub from hanging on ACK.

### spoke_received_network_packet
- **Signature:** `void spoke_received_network_packet(DDPPacketBufferPtr inPacket)`
- **Purpose:** Entry point for incoming packets; validate CRC and dispatch by packet type.
- **Inputs:** Incoming network datagram
- **Outputs/Return:** None (void)
- **Side effects:** Validates CRC, calls packet handler, may update connection state
- **Calls:** `calculate_data_crc_ccitt()`, `spoke_received_game_data_packet_v1()`, logging
- **Notes:** Silently ignores packets if not connected or spoke not active. Catches all exceptions.

### spoke_received_game_data_packet_v1
- **Signature:** `static void spoke_received_game_data_packet_v1(AIStream& ps, bool reflected_flags)`
- **Purpose:** Parse ACK, process messages, enqueue action flags from hub packet.
- **Inputs:** Stream (post-header), reflected-flags flag (true for F1 packet)
- **Outputs/Return:** None (void)
- **Side effects:** Dequeues acknowledged ticks from `sOutgoingFlags`; enqueues to player queues; updates latency measurement; updates timing state
- **Calls:** `process_messages()`, `sNthElementFinder.insert()`, logging
- **Notes:** Handles "we are alone" case (enqueues synthetic net-dead flags). Measures latency as `(write_tick - smallest_unread_tick)`. Skips NthElementFinder on Mac OS 9 for interrupt-safety.

### spoke_tick
- **Signature:** `static bool spoke_tick()`
- **Purpose:** Periodic timer callback (~33ms); generate local flags, apply timing adjustment, send packets.
- **Inputs:** None
- **Outputs/Return:** `bool` true (always reschedule)
- **Side effects:** Increments ticker; may enqueue flags via `parse_keymap()`; sends packets; detects timeout
- **Calls:** `parse_keymap()`, `send_packet()`, `send_identification_packet()`, `check_send_packet_to_hub()`, logging
- **Notes:** Pre-connection: sends ID packets every ~30 ticks. Disconnects if silent for too long. Applies timing adjustment (negative: provide extra flags; positive: skip flags). Manually enqueues net-dead flags when disconnected.

### send_packet
- **Signature:** `static void send_packet()`
- **Purpose:** Build and transmit game data packet to hub (ACK, messages, action flags).
- **Inputs:** None (reads static state)
- **Outputs/Return:** None (void)
- **Side effects:** Builds frame in `sOutgoingFrame`, dequeues lossy descriptors, updates `sLastNetworkTickSent`, sends via network or local hub
- **Calls:** `calculate_data_crc_ccitt()`, `NetDDPSendFrame()` or `send_frame_to_local_hub()`, logging
- **Notes:** Packet format: magic (2B) | CRC (2B) | ACK | optional lossy message | end-of-messages | optional action flags. Includes at most one lossy chunk per packet for robustness.

### send_identification_packet
- **Signature:** `static void send_identification_packet()`
- **Purpose:** Send pre-connection ID packet so hub can map datagram source to player index (NAT-friendly).
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Sends packet via network or local hub
- **Calls:** `calculate_data_crc_ccitt()`, `NetDDPSendFrame()` or `send_frame_to_local_hub()`
- **Notes:** Sent repeatedly until `sHeardFromHub` becomes true.

### spoke_distribute_lossy_streaming_bytes, spoke_distribute_lossy_streaming_bytes_to_everyone
- **Signature:** `void spoke_distribute_lossy_streaming_bytes(int16 type, uint32 bitmask, byte* data, uint16 len)` and variant with team filtering
- **Purpose:** Queue outgoing lossy streaming data for hub to distribute.
- **Inputs:** Distribution type, destinations (bitmask or "everyone"), payload, length
- **Outputs/Return:** None (void)
- **Side effects:** Enqueues descriptor and data to ring buffers; silently discards if full
- **Calls:** Logging
- **Notes:** Supports team-based filtering in the "to_everyone" variant. Logs warning if buffer full.

### spoke_get_net_time
- **Signature:** `int32 spoke_get_net_time()`
- **Purpose:** Return current network time (used for game synchronization).
- **Inputs:** None
- **Outputs/Return:** Tick value
- **Side effects:** May log latency changes
- **Calls:** Logging
- **Notes:** Returns `write_tick - latency` if connected and timing valid; otherwise local queue's write tick.

### Message handlers (handle_end_of_messages_message, handle_player_net_dead_message, handle_timing_adjustment_message, handle_lossy_byte_stream_message)
- **Purpose:** Parse and apply specific message types from hub.
- **Inputs/Outputs:** Stream positioned at message body, context struct
- **Side effects:** Update connection state, timing adjustment, or invoke app callback for lossy data
- **Calls:** `call_distribution_response_function_if_available()` (external), logging

### XML configuration (DefaultSpokePreferences, WriteSpokePreferences, XML_SpokeConfigurationParser)
- Parse/write `<spoke>` XML element with attributes for timeouts, periods, timing parameters; validate ranges

## Control Flow Notes

**Phases:**
1. **Initialization** (`spoke_initialize`): Set up queues, start timer, register handlers.
2. **Pre-connection** (`spoke_tick` sends ID packets every ~30 ticks until `sHeardFromHub`).
3. **Connected** (after hub responds): Exchange action flags and messages via `spoke_received_game_data_packet_v1()` and `spoke_tick()` ΓåÆ `send_packet()`.
4. **Timing adjustment loop**: Hub sends adjustment; spoke applies by skipping or generating extra local ticks.
5. **Disconnection** (timeout): `spoke_tick()` detects silence, calls `spoke_became_disconnected()`, manually enqueues net-dead flags.
6. **Shutdown** (`spoke_cleanup`): Stop timer, clear state.

## External Dependencies
- **Headers:**
  - `network_star.h` ΓÇö Protocol constants, other star functions
  - `AStream.h` ΓÇö Big-endian/little-endian serialization (AIStreamBE, AOStreamBE)
  - `mytm.h` ΓÇö Timer task setup/removal and mutex
  - `network_private.h` ΓÇö `kPROTOCOL_TYPE`, `NET_DEAD_ACTION_FLAG`
  - `WindowedNthElementFinder.h` ΓÇö Latency measurement filter
  - `vbl.h` ΓÇö `parse_keymap()`
  - `CircularByteBuffer.h`, `Logging.h`, `crc.h`, `player.h`, `<map>`
- **External functions:**
  - `make_player_really_net_dead()`, `call_distribution_response_function_if_available()` ΓÇö App callbacks
  - `NetDDP*`, `myXTMSetup()`, `myTMRemove()`, `myTMCleanup()`, mutex functions ΓÇö Platform/network layer
- **Platform:** Guarded by `#if !defined(DISABLE_NETWORKING)`; uses SDL networking indirectly.
