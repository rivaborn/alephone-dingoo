# Subsystem Overview

## Purpose
Manages multiplayer networking for the Aleph One engine, enabling player gathering (hosting), joining, and synchronized gameplay over TCP/UDP connections. Supports both ring and star network topologies, service discovery, game-data distribution, optional voice communication, and metaserver integration for internet play.

## Key Files

| File | Role |
|------|------|
| network.h | Public API for game integration; player gathering/joining, state sync, distribution callbacks |
| network.cpp | Core multiplayer state machine; topology initialization, player lifecycle, message dispatch |
| network_messages.h/cpp | TCPMess protocol: serialization/deserialization of setup/sync messages (HelloMessage, TopologyMessage, etc.) |
| network_private.h | Internal structures: packet headers, ring protocol formats, topology state, error codes |
| RingGameProtocol.h/cpp | Ring-topology implementation: daisy-chain packet forwarding, exponential-backoff retransmission, adaptive latency |
| StarGameProtocol.h/cpp | Star-topology glue layer: hub-and-spoke relay coordination |
| network_star.h | Star protocol interface: hub/spoke action-flag queuing, timing adjustment |
| network_star_hub.cpp | Hub (server): aggregates action flags, detects disconnects, broadcasts to spokes |
| network_star_spoke.cpp | Spoke (client): maintains hub connection, sends/receives action flags, CRC verification |
| network_games.h/cpp | Game-mode-specific rules (King of Hill, Capture Flag, etc.); ranking calculations, end conditions |
| network_data_formats.h/cpp | Cross-platform wire-format structures and endianness conversion via `netcpy()` |
| network_capabilities.h/cpp | Capability versioning for protocol negotiation (gameworld, Lua, Speex, compression) |
| ConnectPool.h/cpp | Non-blocking TCP connection pooling via SDL threads for outbound connects |
| CommunicationsChannel.h/cpp | TCP message buffering and delivery abstraction |
| network_lookup_sdl.h/cpp | SSLP-based network service discovery for LAN games |
| SSLP_*.h/cpp | Simple Service Location Protocol: UDP broadcast service advertisement and discovery |
| network_dialogs.h/cpp | Abstract dialog layer for gather/join UI; SDLNetworkDialog subclass |
| network_dialog_widgets_sdl.h/cpp | Custom widgets: player discovery list, in-game player display, entry-point selector |
| metaserver/network_metaserver.h/cpp | Metaserver client for internet games: login, game listing, chat |
| metaserver/metaserver_messages.h/cpp | Metaserver protocol: serialization of player/game/chat messages |
| metaserver/metaserver_dialogs.h/cpp | Metaserver UI: game browser, player chat |
| network_audio_shared.h | Audio packet header format (sample rate, flags, mono/stereo) |
| network_microphone_shared.h/cpp | Platform-independent audio capture: resampling, Speex encoding, packetization |
| network_speaker_*.cpp | Platform-specific playback: Mac Sound Manager, SDL+ALSA, DirectSound, dummy |
| network_speex.h/cpp | Speex encoder/decoder lifecycle and preprocessing (AGC, denoise) |
| SDL_netx.h/cpp | SDL_net extension for UDP broadcast support |
| network_udp.cpp | Cross-platform UDP via SDL_net; packet reception in dedicated thread |
| network_dummy.cpp | Stub implementations when DISABLE_NETWORKING is set |
| Update.h/cpp | Background thread for online update checking via HTTP |

## Core Responsibilities

- **Topology management**: Initialize and maintain ring or star network topologies; elect new hub on disconnection
- **Player lifecycle**: Gather (server-side) and join (client-side) state machines; drop detection via timeout
- **Message dispatch**: Route incoming TCP/UDP packets to handlers (HelloMessage, TopologyMessage, ActionMessage, etc.)
- **Action flag synchronization**: Queue and broadcast player input across network each game tick; predict unconfirmed flags
- **Game-data distribution**: Broadcast maps, physics, Lua scripts (optionally compressed) to joiners
- **Timing adjustment**: Measure latency; adjust playback delay to tolerate network jitter
- **Service discovery**: Advertise/locate servers on LAN (SSLP) and internet (metaserver)
- **Chat routing**: Relay player and team chat; manage ignore lists
- **Optional voice communication**: Capture microphone input, compress with Speex, stream over network; play back received audio
- **Data format conversion**: Serialize/deserialize all network structures with cross-platform endianness handling
- **Connection pooling**: Non-blocking TCP connects for gatherer/joiner negotiation
- **Capability negotiation**: Exchange protocol versions for Lua, Speex, compression support

## Key Interfaces & Data Flow

**Exposes to game code (network.h):**
- `NetEnter()` / `NetExit()` ΓÇô network subsystem lifecycle
- `NetGather()` / `NetJoin()` ΓÇô multiplayer session entry points
- `NetStart()` ΓÇô begin synchronized gameplay
- `NetDistributeInformation()` ΓÇô broadcast game data (lossy or lossless)
- `NetGetNumberOfPlayers()`, `NetGetPlayerData()` ΓÇô query player state
- Callback registration: `gatherCallbacks` (join/drop notifications), `chatCallbacks`, `checkPlayer`

**Consumes from game:**
- `dynamic_world` ΓÇô current game state (cheat flags, player count)
- `process_action_flags()` ΓÇô deliver synchronized action flags each tick
- `GetRealActionQueues()` ΓÇô action flag input from local player
- `make_player_really_net_dead()` ΓÇô mark disconnected players as inactive
- Player/scenario data via `player_data`, `Scenario::instance()`

**Consumes from network:**
- UDP/TCP packets from remote players (via `NetDDPOpenSocket`, `SDLNet_UDP_Open`)
- Metaserver responses (game listings, chat) via TCP to metaserver
- SSLP discovery packets (service broadcasts) via UDP

**Distributes to game:**
- Remote player action flags via `process_action_flags()`
- Chat messages via callback
- Audio samples (for voice playback) via `Mixer` singleton or Sound Manager

## Runtime Role

**Initialization (pre-game):**
- Call `NetEnter()` to enable networking
- Call `NetGather()` (host) or `NetJoin()` (joiner) to enter lobby
- `network_dialogs.cpp` runs dialog event loop; calls network pump functions
- Metaserver and SSLP discovery active during lobby

**Frame/per-tick (during game):**
- Ring protocol: local player queues action flags; on each frame, sends packet upring
- Star protocol hub: receives action flags from all spokes; broadcasts aggregated flags each tick
- Star protocol spoke: sends action flags to hub; receives other players' flags from hub
- Voice capture: microphone thread continuously buffered; transmitted in ~20ms chunks
- Voice playback: decode incoming audio packets and feed to speaker/mixer
- Timing adjustment: measure latency, adjust playback lead-time if drift detected

**Shutdown:**
- Call `NetExit()` to close topology, cleanup threads, deallocate buffers
- Disconnect from metaserver if connected

## Notable Implementation Details

- **Ring vs. Star trade-off**: Ring is lower-bandwidth (peer-to-peer) but higher-latency (packet chains). Star is lower-latency (hub relay) but higher bandwidth (hub forwards to all).
- **Lossy retransmission**: Ring protocol uses exponential-backoff retransmit on loss detection; Star hub tracks per-spoke smallest-unacknowledged-tick.
- **Compression**: Large payloads (maps, physics, Lua scripts) compressed with zlib; metaserver messages deflated via AIStream/AOStream.
- **Voice codec**: Speex narrowband (8 kHz) with quality=3, complexity=4; auto-gain-control and denoise preprocessing.
- **Service discovery**: SSLP is a custom simplified protocol (5-second discovery interval, best-effort). Metaserver requires explicit server address.
- **Platform-specific audio**: Sound Manager (Mac classic), CoreAudio (macOS), ALSA (Linux), DirectSound (Windows), dummy (no-op) for unsupported platforms.
- **Dingoo-specific**: Audio subsystems likely disabled (HAVE_DINGOO guard); voice chat probably not feasible on 400 MHz MIPS with 32 MB RAM. Core TCP/UDP networking and ring/star protocols remain; metaserver support may be reduced or disabled depending on compile configuration.
