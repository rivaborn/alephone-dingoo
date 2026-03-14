# Source_Files/Network/RingGameProtocol.cpp

## File Purpose
Implementation of a ring-topology network protocol for multiplayer games. Manages packet routing through a logical ring of connected players, handling action flag distribution, acknowledgments, retransmissions, and adaptive latency adjustment to optimize responsiveness vs. smoothness under varying network conditions.

## Core Responsibilities
- Initialize/finalize network connections and frame allocation
- Synchronize players at game startup; unsync at shutdown
- Queue local action flags and transmit through the ring
- Handle incoming ring packets and process action flags for all players
- Manage acknowledgments and implement exponential-backoff retransmission
- Elect new server if upring player disconnects
- Implement three variants of adaptive latency to tolerate jitter
- Maintain ring topology (upring/downring neighbor addresses)
- Parse XML configuration for ring protocol settings

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NetStatus` | struct | Tracks network state: addresses, sequence numbers, packet counts, queue refs, latency settings |
| `NetQueue` | struct | Circular buffer of queued action flags |
| `RingPreferences` | struct | Configurable behavior: accept packets from anyone, adapt to latency, latency hold duration |
| `XML_RingConfigurationParser` | class | Parses `<ring_protocol>` XML elements and updates `sRingPreferences` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `status` | `NetStatusPtr` | static volatile | Main network state; malloc'd on `Enter()` |
| `ringFrame`, `ackFrame`, `distributionFrame` | `DDPFramePtr` | static | DDP frames for outgoing packets (allocated on `Enter()`) |
| `local_queue` | `NetQueue volatile` | static | Circular buffer of locally queued action flags |
| `unpackedReceiveBuffer` | `char[ddpMaxData]` | static | Temporary unpacked packet buffer |
| `sRingPreferences` | `RingPreferences` | static | Parsed XML config (accept packets from anyone, adapt latency, hold ticks) |
| `resendTMTask`, `serverTMTask`, `queueingTMTask` | `myTMTaskPtr` | static | Scheduled tasks (retransmit, server periodic, client queuing) |
| `sAdaptiveLatencySamples[]`, `sCurrentAdaptiveLatency`, etc. | `int[kAdaptiveLatencyWindowSize]`, `int volatile` | static | Adaptive latency state (multiple impls active per compile flags) |
| `sFlagDitchingCounter` | `int` | static | Flag ditching counter (NETWORK_SMARTER_FLAG_DITCHING only) |
| `localPlayerIndex`, `topology`, `ddpSocket` | `short`, `NetTopology*`, `short` | static | Local player index, ring topology, dummy socket |

## Key Functions / Methods

### `Enter(short* inNetStatePtr)`
- **Signature:** `bool Enter(short* inNetStatePtr)`
- **Purpose:** Initialize the ring protocol module; allocate status structure, buffers, and DDP frames.
- **Inputs:** Pointer to net state variable (to be updated by packet handler).
- **Outputs/Return:** `true` if all allocations succeed, `false` otherwise.
- **Side effects:** Allocates heap memory (`status`, `status->buffer`, frames); registers packet handler callback.
- **Calls:** `malloc()`, `NetDDPNewFrame()`, `memset()`.
- **Notes:** Called once at the start of networking. Initializes debug stats if `DEBUG_NET` enabled.

### `Exit1()`
- **Signature:** `void Exit1()`
- **Purpose:** Remove and clean up scheduled tasks (before Exit2 frees frames).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `myTMRemove()` on three timer tasks (safe for `NULL`).
- **Calls:** `myTMRemove()`.
- **Notes:** Separate from `Exit2()` to allow task removal before frame deallocation.

### `Exit2()`
- **Signature:** `void Exit2()`
- **Purpose:** Free heap-allocated resources (buffers, frames, status).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `free()`, `NetDDPDisposeFrame()`, sets `status` to `NULL`.
- **Calls:** `free()`, `NetDDPDisposeFrame()`, debug close-stream.
- **Notes:** Called after `Exit1()`. Inverse of `Enter()`.

### `Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex)`
- **Signature:** `bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex)`
- **Purpose:** Synchronize all players at game start by waiting for the ring to circulate twice. Computes upring/downring neighbors, starts the ring if server, and polls until `netState` becomes `netActive`.
- **Inputs:** Topology, smallest game tick, local player index, server player index.
- **Outputs/Return:** `true` if sync succeeded, `false` on timeout or error.
- **Side effects:** Updates `status` fields (addresses, latency params, queue indices); if server, creates first ring packet and sends; schedules `serverTMTask` or `queueingTMTask`; alerts user on timeout.
- **Calls:** `NetBuildFirstRingPacket()`, `NetSendRingPacket()`, `machine_tick_count()`, `myXTMSetup()`, `alert_user()`.
- **Notes:** Blocks until packet handler changes `*sNetStatePtr` to `netActive`. Skips players with `identifier == NONE` (zombie slots).

### `UnSync(bool inGraceful, int32 inSmallestPostgameTick)`
- **Signature:** `bool UnSync(bool inGraceful, int32 inSmallestPostgameTick)` (declaration only in header; body truncated in file)
- **Purpose:** Clean exit from network game; ensures no player holds the ring token.
- **Inputs:** Graceful flag, smallest postgame tick.
- **Outputs/Return:** Boolean indicating success.
- **Side effects:** (Truncated in file) Sets `netState` to `netComingDown`, waits for unsync packets.
- **Notes:** Complementary to `Sync()`.

### `GetNetTime()`
- **Signature:** `int32 GetNetTime(void)`
- **Purpose:** Return the effective network time for the game engine, adjusted for adaptive latency.
- **Inputs:** None.
- **Outputs/Return:** `status->localNetTime` minus an adaptive latency offset (if adaptive latency enabled); else minus fixed offset.
- **Side effects:** None.
- **Calls:** None directly.
- **Notes:** The offset masks jitter by holding the game behind the actual net time. Compile-time selection between three adaptive latency schemes or a static offset.

### `GetUnconfirmedActionFlagsCount()`, `PeekUnconfirmedActionFlag()`, `UpdateUnconfirmedActionFlags()`
- **Signature:** `int32 GetUnconfirmedActionFlagsCount()`, `uint32 PeekUnconfirmedActionFlag(int32 offset)`, `void UpdateUnconfirmedActionFlags()`
- **Purpose:** Query the local action queue for prediction/speculative updates (spoke interface).
- **Inputs:** Offset for peek.
- **Outputs/Return:** Count of unconfirmed flags; flag at offset; none.
- **Side effects:** None (query-only).
- **Calls:** `GetRealActionQueues()`.
- **Notes:** Used by client-side prediction code.

### `update_adaptive_latency(int measurement, int tick)`
- **Signature:** `static void update_adaptive_latency(int measurement = 0, int tick = 0)`
- **Purpose:** Adjust `sCurrentAdaptiveLatency` based on measured ring latency (NETWORK_ADAPTIVE_LATENCY_2/3 only).
- **Inputs:** Measured latency (rings per tick), current tick count.
- **Outputs/Return:** None.
- **Side effects:** Updates `sCurrentAdaptiveLatency` (used by `GetNetTime()`), sample buffer, etc.
- **Calls:** `logNote1()`, `logDump2()`.
- **Notes:** Three implementations: (1) average-based, (2) based on required flags, (3) peak-with-falloff. Only one active per compile config.

### `NetAdjustUpringAddressUpwards()`
- **Signature:** `static short NetAdjustUpringAddressUpwards(void)`
- **Purpose:** Update `status->upringAddress` to the next player and return the old upring player index.
- **Inputs:** None (uses global `topology`, `status`).
- **Outputs/Return:** Previous upring player index.
- **Side effects:** Modifies `status->upringAddress`, `status->upringPlayerIndex`; skips dead players.
- **Calls:** None.
- **Notes:** Helper for dynamic ring rearrangement.

### `drop_upring_player()`
- **Signature:** `static void drop_upring_player(void)`
- **Purpose:** Handle upring player disconnection: unpack ring frame, mark player dead, shift action flags, find new upring, possibly elect new server.
- **Inputs:** None (uses global `ringFrame`, `status`, `topology`).
- **Outputs/Return:** None.
- **Side effects:** Modifies packed ring frame; may set `status->iAmTheServer = true` and schedule `serverTMTask`; marks player net-dead.
- **Calls:** `netcpy()`, `NetAdjustUpringAddressUpwards()`, `myTMRemove()`, `myXTMSetup()`, `memmove()`, `NetBuildRingPacket()`, `NetRebuildRingPacket()`.
- **Notes:** Uses `memmove()` instead of `memcpy()` for safe overlapping buffer copy.

### `process_packet_buffer_flags(void* buffer, int32 buffer_size, short packet_tag)`
- **Signature:** `static void process_packet_buffer_flags(void* buffer, int32 buffer_size, short packet_tag)`
- **Purpose:** Process received ring packet: unpack, call `process_flags()`, add local flags, build/send outgoing ring packet or buffer for server task.
- **Inputs:** Unpacked buffer, size, packet tag (NONE or tagCHANGE_RING_PACKET).
- **Outputs/Return:** None.
- **Side effects:** Modifies `ringFrame`, sends packets, updates `status->canForwardRing`, updates adaptive latency, may change `netState`.
- **Calls:** `process_flags()`, `NetAddFlagsToPacket()`, `NetBuildRingPacket()`, `NetRebuildRingPacket()`, `NetSendAcknowledgement()`, `NetSendRingPacket()`, `update_adaptive_latency()`, `memcpy()`.
- **Notes:** Different control paths for server vs. non-server and for ring-ready vs. buffered cases.

### `process_flags(NetPacket* packet_data)`
- **Signature:** `static void process_flags(NetPacket* packet_data)`
- **Purpose:** Iterate over all players in packet and call `process_action_flags()` for each player's action flags (or inject dead flags if player is net-dead).
- **Inputs:** Unpacked packet.
- **Outputs/Return:** None.
- **Side effects:** Calls game engine's `process_action_flags()` to apply actions to game state.
- **Calls:** `process_action_flags()` (defined elsewhere), optional debug streaming.
- **Notes:** Handles variable action flag counts per player; marks dead players.

### `NetSizeofLocalQueue()`
- **Signature:** `static short NetSizeofLocalQueue(void)`
- **Purpose:** Compute circular buffer size of `local_queue`.
- **Inputs:** None.
- **Outputs/Return:** Number of queued flags.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Helper; wraps around if read > write index.

### `NetPrintInfo()`
- **Signature:** `void NetPrintInfo(void)`
- **Purpose:** Print debug statistics (packet counts, retries, late packets, ACKs, etc.) to debug output.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `fdprintf()` (debug output).
- **Calls:** `fdprintf()`.
- **Notes:** `DEBUG_NET` only.

### `RingGameProtocol::GetParser()`
- **Signature:** `XML_ElementParser* RingGameProtocol::GetParser()`
- **Purpose:** Return the XML parser for `<ring_protocol>` config elements.
- **Inputs:** None.
- **Outputs/Return:** Pointer to static `RingConfigurationParser`.
- **Side effects:** None.
- **Calls:** None.

### `WriteRingPreferences(FILE* F)`, `DefaultRingPreferences()`
- **Signature:** `void WriteRingPreferences(FILE* F)`, `void DefaultRingPreferences()`
- **Purpose:** Write preferences to file (XML format); set defaults.
- **Inputs:** File pointer for write; none for defaults.
- **Outputs/Return:** None.
- **Side effects:** `fprintf()` to file; updates `sRingPreferences` to defaults.
- **Calls:** `fprintf()`.

## Control Flow Notes
- **Init:** `Enter()` ΓåÆ `Sync()` allocates resources, computes topology, starts ring if server.
- **Frame:** Scheduled tasks (`NetServerTask`, `NetQueueingTask`) run periodically, queuing/processing actions. Packet handler processes incoming packets asynchronously.
- **Shutdown:** `UnSync()` ΓåÆ `Exit1()` / `Exit2()` clean up.
- **Adaptive latency:** Updated when ring packets arrive (in `process_packet_buffer_flags`), affects `GetNetTime()` return value to delay engine's consumption of action flags.

## External Dependencies
- **Includes:** `config.h`, `cseries.h`, `Carbon.h` (Mac), `ActionQueues.h`, `player.h`, `network.h`, `network_private.h`, `network_data_formats.h`, `mytm.h`, `map.h`, `vbl.h`, `interface.h`, `XML_ElementParser.h`, `Logging.h`
- **External symbols:** `GetRealActionQueues()`, `process_action_flags()` (interface.h), `NetDDPNewFrame()`, `NetDDPDisposeFrame()`, `NetDDPSendFrame()` (sdl_network.h), `myTMRemove()`, `myXTMSetup()` (mytm.h), `alert_user()`, `dynamic_world` (game state), `machine_tick_count()` (timing), `netcpy()` (network_data_formats.h), `logNote1()`, `logDump2()` (Logging.h)
- **Conditionally enabled:** Multiple adaptive latency schemes; debug statistics and streaming (STREAM_NET); modem sync fallback.
