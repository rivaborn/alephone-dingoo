# Source_Files/Network/network.cpp

## File Purpose
Core network protocol implementation for Aleph One multiplayer games. Manages player joining/gathering, network state synchronization, topology distribution, game-data broadcasting, and message-based communication between gatherers (servers) and joiners (clients). Supports both Ring and Star game protocols with adaptive latency and compression.

## Core Responsibilities
- Player gathering (server-side) and joining (client-side) state machines
- Network topology initialization and distribution
- Connection lifecycle management (client connections, disconnections, drops)
- Game-data distribution (maps, physics models, Lua scripts) with optional compression
- Chat message relaying and player ignore-list management
- Message dispatching for protocol-specific handlers (join, capabilities, topology, etc.)
- Network statistics collection and transmission
- Integration with game protocol layer (Ring/Star protocols)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Client | class | Represents a remote player connection with state machine, message handlers, and capabilities. |
| NetTopology | struct (defined elsewhere) | Stores current network game topology (players, identifiers, addresses). |
| TopologyMessage | class | Serialized network topology for distribution to players. |
| JoinerInfoMessage, CapabilitiesMessage, AcceptJoinMessage, etc. | classes | Protocol messages for handshake and game setup. |
| NetworkGameProtocol | abstract base class | Interface for Ring/Star protocol implementations. |
| Capabilities | struct | Tracks client capabilities (Lua, Star, Ring, zipped data, etc.). |
| NetworkStats | struct | Network statistics (latency, packet loss). |
| distribution_info_map_t | std::map | Maps distribution types to lossy flags and handling procedures. |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| ddpSocket | short | static | UDP socket number for DDP frames. |
| localPlayerIndex, localPlayerIdentifier | short | static | Local player's position in topology and unique ID. |
| topology | NetTopologyPtr | static | Current network topology (all players, game data). |
| sCurrentGameProtocol | NetworkGameProtocol* | static | Active protocol instance (Ring or Star). |
| sRingGameProtocol, sStarGameProtocol | RingGameProtocol, StarGameProtocol | static | Protocol instances. |
| netState | short | static | Network state machine (netUninitialized, netGathering, netJoining, etc.). |
| connections_to_clients | std::map<int, Client*> | static | Active client connections indexed by stream ID (gatherer only). |
| client_chat_info | std::map<int, ClientChatInfo*> | static | Chat info for connected clients (names, colors, teams). |
| connection_to_server | CommunicationsChannel* | static | Connection to gatherer (joiner only). |
| deferred_script_data, deferred_script_length | byte*, size_t | static | Lua script pending distribution. |
| do_netscript | bool | static | Whether to distribute Lua scripts. |
| host_address | IPaddress | static | Target gatherer address (joiner). |
| sIgnoredPlayers | std::set<int> | static | Ignored player indices (for chat filtering). |
| sNetworkStats | std::vector<NetworkStats> | static | Per-player network statistics (Star protocol). |

## Key Functions / Methods

### Client constructor
- **Signature:** `Client::Client(CommunicationsChannel *inChannel)`
- **Purpose:** Initialize a client connection with message dispatchers for join, capabilities, chat, color-change, and accept-join messages.
- **Inputs:** CommunicationsChannel pointer to remote connection.
- **Outputs/Return:** None (constructor).
- **Side effects:** Allocates MessageDispatcher, registers message handlers, initializes name buffer.
- **Calls:** newMessageHandlerMethod() (template), mDispatcher->setDefaultHandler(), mDispatcher->setHandlerForType().
- **Notes:** One Client per remote player during gathering phase.

### Client::drop
- **Signature:** `void Client::drop()`
- **Purpose:** Cleanly disconnect a client (drop from pre-game or mid-game), remove from topology, and notify remaining clients.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Removes from topology/chat info, broadcasts ClientInfoMessage::kRemove, triggers callbacks (JoiningPlayerDropped, JoinedPlayerDropped), updates topology.
- **Calls:** getStreamIdFromChannel(), client_chat_info access, NetUpdateTopology(), NetDistributeTopology(), callbacks.
- **Notes:** Handles two states: _connected (pre-game) and _awaiting_map (mid-game). Logs anomaly if state is unexpected.

### Client::capabilities_indicate_player_is_gatherable
- **Signature:** `bool Client::capabilities_indicate_player_is_gatherable(bool warn_joiner)`
- **Purpose:** Validate client capabilities against gatherer requirements (Lua version, protocol support, etc.).
- **Inputs:** warn_joinerΓÇöif true, send ServerWarningMessage to client.
- **Outputs/Return:** boolΓÇötrue if gathereable, false if incompatible.
- **Side effects:** Sends ServerWarningMessage if warn_joiner=true and incompatible.
- **Calls:** getcstr() (string lookup), channel->enqueueOutgoingMessage().
- **Notes:** Checks kGatherable, kStar/kRing (protocol-dependent), kLua (if netscript enabled). Logs no warning if client self-disabled via Gatherable=0.

### Client::handleJoinerInfoMessage
- **Signature:** `void Client::handleJoinerInfoMessage(JoinerInfoMessage* joinerInfoMessage, CommunicationsChannel *)`
- **Purpose:** Process joiner introduction (name, color, team); set up chat info, broadcast to other clients, respond with gatherer capabilities.
- **Inputs:** JoinerInfoMessage with joiner's name, color, team.
- **Outputs/Return:** None.
- **Side effects:** Updates client name, creates ClientChatInfo, enqueues ClientInfoMessage to other clients, changes state to _awaiting_capabilities, sends CapabilitiesMessage.
- **Calls:** pstrcpy(), a1_p2cstr(), getStreamIdFromChannel(), new ClientChatInfo, enqueueOutgoingMessage().
- **Notes:** Only valid in netGathering state; disconnects client (_disconnect) if version mismatch.

### Client::handleCapabilitiesMessage
- **Signature:** `void Client::handleCapabilitiesMessage(CapabilitiesMessage* capabilitiesMessage, CommunicationsChannel *)`
- **Purpose:** Receive client capabilities, validate against gatherer, update state (connected_but_not_yet_shown or ungatherable), broadcast existing client info.
- **Inputs:** CapabilitiesMessage.
- **Outputs/Return:** None.
- **Side effects:** Updates client->capabilities, changes state, enqueues ClientInfoMessage to the newly-connected client for all existing clients in chat.
- **Calls:** capabilities_indicate_player_is_gatherable(), enqueueOutgoingMessage().
- **Notes:** Gates gathering: if client cannot be gathered, state becomes _ungatherable and client is not shown in join list.

### Client::handleAcceptJoinMessage
- **Signature:** `void Client::handleAcceptJoinMessage(AcceptJoinMessage* acceptJoinMessage, CommunicationsChannel *)`
- **Purpose:** Add accepted joiner to topology, assign stream_id and address, broadcast topology update, transition to _awaiting_map.
- **Inputs:** AcceptJoinMessage with joiner's player data.
- **Outputs/Return:** None.
- **Side effects:** Adds to topology->players[], increments topology->player_count, calls check_player callback, updates topology distribution, changes state to _awaiting_map, triggers gatherCallbacks->JoinSucceeded.
- **Calls:** getStreamIdFromChannel(), check_player(), NetUpdateTopology(), NetDistributeTopology(tagNEW_PLAYER).
- **Notes:** Joiner must have accepted join (acceptJoinMessage->accepted()==true); otherwise state becomes _ungatherable and error alert is shown.

### Client::handleChangeColorsMessage
- **Signature:** `void Client::handleChangeColorsMessage(ChangeColorsMessage *changeColorsMessage, CommunicationsChannel *channel)`
- **Purpose:** Relay color/team change during pre-game or in-game, update topology if needed, broadcast updates.
- **Inputs:** ChangeColorsMessage with new color and team.
- **Outputs/Return:** None.
- **Side effects:** Updates client_chat_info and/or topology->players[], broadcasts ClientInfoMessage or TopologyMessage, calls check_player() and callback.
- **Calls:** getStreamIdFromChannel(), enqueueOutgoingMessage(), NetUpdateTopology(), NetDistributeTopology().
- **Notes:** Handles both pre-game (_connected) and in-game (_awaiting_map) states; ignores if client is not in pre-game chat state (unless _awaiting_map).

### NetGatherPlayer
- **Signature:** `int NetGatherPlayer(const prospective_joiner_info &player, CheckPlayerProcPtr check_player)`
- **Purpose:** Accept a prospective joiner and transition to awaiting acceptance.
- **Inputs:** prospective_joiner_info (stream_id, name, etc.), check_player callback.
- **Outputs/Return:** kGatherPlayerSuccessful.
- **Side effects:** Enqueues JoinPlayerMessage to joiner's channel, updates Client state to _awaiting_accept_join, stores check_player callback.
- **Calls:** connections_to_clients[stream_id]->channel->enqueueOutgoingMessage().
- **Notes:** Asserts netState == netGathering and player_count < MAXIMUM_NUMBER_OF_NETWORK_PLAYERS.

### NetUpdateJoinState
- **Signature:** `short NetUpdateJoinState(void)`
- **Purpose:** Joiner-side state machine: transition from Connecting ΓåÆ Joining ΓåÆ Waiting ΓåÆ (game start handled elsewhere).
- **Inputs:** None (uses static handlerState, connection_to_server).
- **Outputs/Return:** newState (netConnecting, netJoining, netWaiting, netJoinErrorOccurred, etc.).
- **Side effects:** Pumps connection_to_server, dispatches incoming messages, updates handlerState, may transition netState. Polls NonblockingConnect for connection status every 5 seconds (configurable backoff).
- **Calls:** connection_to_server->pump(), dispatchIncomingMessages(), NonblockingConnect::status(), ConnectPool operations, alert_user().
- **Notes:** Non-blocking; retries connection on failure. Returns netPlayerAdded, netPlayerDropped, netPlayerChanged, etc. as signals without changing netState.

### NetDistributeGameDataToAllPlayers
- **Signature:** `OSErr NetDistributeGameDataToAllPlayers(byte *wad_buffer, int32 wad_length, bool do_physics)`
- **Purpose:** Broadcast map, physics, and Lua script data to all non-local, non-dead players, with compression support and progress tracking.
- **Inputs:** wad_buffer (map data), wad_length, do_physics (whether to include physics).
- **Outputs/Return:** OSErr (0 on success, otherwise error code).
- **Side effects:** Opens progress dialog, splits channels by compression capability, enqueues MapMessage/ZippedMapMessage/PhysicsMessage/etc., flushes outgoing messages, updates Client states to _ingame, processes physics and Lua.
- **Calls:** open_progress_dialog(), std::for_each() with boost::bind, CommunicationsChannel::multipleFlushOutgoingMessages(), process_network_physics_model(), LoadLuaScript(), close_progress_dialog().
- **Notes:** Timeout if transfer exceeds MAP_TRANSFER_TIME_OUT * player_count ticks. Adaptive: sends zipped data to capable clients, uncompressed to others.

### NetReceiveGameData
- **Signature:** `byte *NetReceiveGameData(bool do_physics)`
- **Purpose:** Joiner-side: wait for map, physics, and Lua script from gatherer; block until EndGameDataMessage received.
- **Inputs:** do_physics (whether to process physics).
- **Outputs/Return:** byte* (pointer to map buffer, or NULL if failed).
- **Side effects:** Opens progress dialog, pumps connection_to_server, processes incoming MapMessage/PhysicsMessage/LuaMessage via handlers, closes dialog on completion or timeout, sets global handler buffers (handlerMapBuffer, handlerPhysicsBuffer, handlerLuaBuffer).
- **Calls:** connection_to_server->receiveSpecificMessage<EndGameDataMessage>(), process_network_physics_model(), LoadLuaScript(), close_progress_dialog(), alert_user().
- **Notes:** 60s total timeout, 30s per-message timeout. Returns NULL and displays error if timeout/failure.

### NetCheckForNewJoiner
- **Signature:** `bool NetCheckForNewJoiner(prospective_joiner_info &info)`
- **Purpose:** Accept incoming connections, pump existing clients, and return next prospective joiner ready to display.
- **Inputs:** prospective_joiner_info reference (output).
- **Outputs/Return:** boolΓÇötrue if a new joiner transitioned to _connected_but_not_yet_shown, sets info.
- **Side effects:** Accepts new TCPsocket connections via server->newIncomingConnection(), creates Client, assigns stream_id, sends HelloMessage, pumps all client channels, drops disconnected clients, cleans up dropped clients.
- **Calls:** server->newIncomingConnection(), new Client(), channel->pump(), dispatchIncomingMessages(), Client::drop(), delete.
- **Notes:** Filters joiners by state: only returns _connected_but_not_yet_shown; removes _disconnect and disconnected clients from map.

### NetDDPPacketHandler
- **Signature:** `void NetDDPPacketHandler(DDPPacketBufferPtr inPacket)`
- **Purpose:** Dispatch incoming UDP (DDP) packet to active game protocol.
- **Inputs:** DDPPacketBufferPtr (raw DDP frame).
- **Outputs/Return:** None.
- **Side effects:** Calls sCurrentGameProtocol->PacketHandler().
- **Calls:** sCurrentGameProtocol->PacketHandler().
- **Notes:** Entry point for DDP-level packet handling; delegates to Ring or Star protocol.

### NetInitializeTopology
- **Signature:** `static void NetInitializeTopology(void *game_data, short game_data_size, void *player_data, short player_data_size)`
- **Purpose:** Initialize topology with local player as player 0, set up initial game and player data.
- **Inputs:** game_data (game_info*), game_data_size, player_data (player_info*), player_data_size.
- **Outputs/Return:** None.
- **Side effects:** Initializes topology->players[0], sets localPlayerIndex/localPlayerIdentifier to 0, copies game/player data, sets player_count=1.
- **Calls:** NetLocalAddrBlock(), memcpy().
- **Notes:** Assumes local player is always index 0, identifier 0. NetLocalAddrBlock sets dummy localhost address (0x7f000001).

### NetLocalAddrBlock
- **Signature:** `static void NetLocalAddrBlock(NetAddrBlock *address, short socketNumber)`
- **Purpose:** Fill in a local address block (localhost + socket/port).
- **Inputs:** NetAddrBlock pointer, socket number.
- **Outputs/Return:** None (modifies address in-place).
- **Side effects:** Sets address->host to 0x7f000001 (localhost), address->port to socketNumber.
- **Calls:** None.
- **Notes:** Comment notes this is "really bad" and that addresses are usually overwritten by real values from peers.

### NetUpdateTopology
- **Signature:** `static void NetUpdateTopology(void)`
- **Purpose:** Recalculate localPlayerIndex based on current localPlayerIdentifier.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Scans topology->players, updates localPlayerIndex.
- **Calls:** None.
- **Notes:** Called after topology changes; asserts localPlayerIndex < player_count in DEBUG.

### NetDistributeTopology
- **Signature:** `static void NetDistributeTopology(short tag)`
- **Purpose:** Broadcast topology update to all players except self.
- **Inputs:** tag (topology change reason: tagNEW_PLAYER, tagDROPPED_PLAYER, tagCHANGED_PLAYER).
- **Outputs/Return:** None.
- **Side effects:** Sets topology->tag, enqueues TopologyMessage to all non-local, non-NONE players via connections_to_clients.
- **Calls:** TopologyMessage constructor, channel->enqueueOutgoingMessage().
- **Notes:** Only valid in netGathering state (asserted).

### NetChangeMap
- **Signature:** `bool NetChangeMap(struct entry_point *entry)`
- **Purpose:** Coordinate map change between server and clients: server sends, clients receive.
- **Inputs:** entry_point with level info.
- **Outputs/Return:** boolΓÇötrue on success, false on error (server died, receive timeout).
- **Side effects:** If local player is server, gets map data and distributes via NetDistributeGameDataToAllPlayers(); if not, waits for map via NetReceiveGameData(). Both paths then call process_net_map_data().
- **Calls:** NetDistributeGameDataToAllPlayers(), NetReceiveGameData(), get_map_for_net_transfer(), process_net_map_data(), alert_user().
- **Notes:** Server death during level change causes failure.

### NetProcessMessagesInGame
- **Signature:** `void NetProcessMessagesInGame()`
- **Purpose:** During active game: pump connection to server (joiner) or all client connections (gatherer), relay chat, and collect statistics.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Pumps and dispatches messages on connection_to_server or all client channels, broadcasts NetworkStatsMessage to clients periodically (Star protocol), updates last_network_stats_send.
- **Calls:** connection_to_server->pump(), dispatchIncomingMessages(), client->channel->pump(), dispatchIncomingMessages(), hub_stats(), enqueueOutgoingMessage().
- **Notes:** Gathers stats and sends to clients every MACHINE_TICKS_PER_SECOND ticks (Star protocol).

### NetGetLatency, NetGetStats, NetGetUnconfirmedActionFlagsCount, etc.
- **Signature:** Various query functions (`int32 NetGetLatency()`, `const NetworkStats& NetGetStats(int player_index)`, etc.)
- **Purpose:** Query in-game network metrics (latency, packet loss, pending actions) from protocol layer.
- **Inputs:** Varies (player_index for stats, offset for flags).
- **Outputs/Return:** Varies (int32, const NetworkStats&, uint32).
- **Side effects:** None (read-only).
- **Calls:** sCurrentGameProtocol->GetNetTime(), spoke_latency(), hub_stats(), sCurrentGameProtocol->GetUnconfirmedActionFlagsCount(), PeekUnconfirmedActionFlag(), UpdateUnconfirmedActionFlags().
- **Notes:** Star protocol queries spoke_latency() when connection_to_server present; Ring/unknown returns NetworkStats::invalid.

## Control Flow Notes

**Gatherer (server) flow:**
1. Initialize topology via NetInitializeTopology().
2. Accept connections via NetCheckForNewJoiner(), create Client objects, transition through states (Connecting ΓåÆ AwaitingCapabilities ΓåÆ ConnectedButNotYetShown ΓåÆ Connected).
3. Gather players via NetGatherPlayer(), add to topology, distribute topology updates.
4. Broadcast game data via NetDistributeGameDataToAllPlayers() (map, physics, Lua).
5. During game, pump clients via NetProcessMessagesInGame(), relay chat, collect stats.

**Joiner (client) flow:**
1. Initiate connection via NetUpdateJoinState() state machine (netConnecting ΓåÆ netJoining).
2. Receive HelloMessage, respond with JoinerInfoMessage.
3. Receive CapabilitiesMessage, send JoinerInfoMessage.
4. Receive JoinPlayerMessage, send AcceptJoinMessage.
5. Receive TopologyMessage updates.
6. Receive map, physics, Lua via NetReceiveGameData().
7. During game, pump server connection via NetProcessMessagesInGame().

**Message handlers:**
- Joiner-side: handleHelloMessage, handleCapabilitiesMessage, (in global scope).
- Gatherer-side: Client::handleJoinerInfoMessage, handleCapabilitiesMessage, handleAcceptJoinMessage, handleChangeColorsMessage, handleChatMessage.

## External Dependencies
- **Notable includes:**
  - `map.h`: TICKS_PER_SECOND, entry_point struct.
  - `interface.h`: MAP transfer functions (get_map_for_net_transfer, process_net_map_data).
  - `mytm.h`: Thread task management.
  - `preferences.h`: network_preferences, player_preferences, environment_preferences.
  - `sdl_network.h`: DDP/ADSP networking primitives (DDPPacketBuffer, CommunicationsChannel).
  - `CommunicationsChannel.h`: TCP message handling and connection pooling.
  - `MessageDispatcher.h`, `MessageInflater.h`, `MessageHandler.h`: Protocol message infrastructure.
  - `NetworkGameProtocol.h`, `RingGameProtocol.h`, `StarGameProtocol.h`: Game protocol implementations.
  - `lua_script.h`: Lua scripting integration (LoadLuaScript).
  - `libnat.h`: UPnP/NAT traversal.
  - `boost/bind.hpp`: Functional programming utilities.
  - `network_metaserver.h`: Metaserver integration (gMetaserverClient).
  - `network_sound.h`: Voice chat (not heavily featured in this file).
  - `ConnectPool.h`: Nonblocking connection pooling.

- **Defined elsewhere (external symbols):**
  - `topology` (NetTopology*): Network topology struct (defined in network_private.h or elsewhere).
  - `dynamic_world`: Game world state (used for cheat flags, player count).
  - `sServerPlayerIndex`: Server player identifier.
  - `check_player`: Callback for validating new players.
  - `gatherCallbacks`, `chatCallbacks`: Game-level callbacks for events.
  - `hub_stats()`: Ring protocol statistics.
  - `spoke_latency()`: Star protocol latency.
  - Various message classes (TopologyMessage, MapMessage, etc.): Defined in network_messages.h and subclasses.
