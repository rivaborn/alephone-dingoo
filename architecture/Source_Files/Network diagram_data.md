# Source_Files/Network/CarbonSndPlayDB.h
## File Purpose
Header file defining Carbon-compatible sound playback structures and APIs. Provides a double-buffered audio playback interface that mimics the legacy `SndPlayDoubleBuffer` but works with the Carbon runtime environment on macOS.

## Core Responsibilities
- Define data structures for double-buffered audio playback
- Declare callback types for buffer completion notifications
- Export public functions for Carbon sound channel operations
- Provide compatibility layer between legacy Sound Manager API and Carbon

## External Dependencies
- Mac Sound Manager types: `SndChannelPtr`, `SndCommand`, `OSErr`, `UnsignedFixed`, `OSType`
- Carbon macros: `CALLBACK_API`, `STACK_UPP_TYPE`, `__BEGIN_DECLS`, `__END_DECLS`
- Conditional: `<sys/cdefs.h>` (CodeWarrior compatibility fallback included)

# Source_Files/Network/ConnectPool.cpp
## File Purpose
Implements a non-blocking TCP connection pool for the Aleph One engine. Manages outbound connection establishment in worker threads, allowing DNS resolution and socket connection without blocking the main thread. Uses a fixed-size pool of reusable connection objects.

## Core Responsibilities
- Spawn and manage worker threads for individual TCP connection attempts
- Perform DNS resolution (hostname ΓåÆ IP) in worker threads via SDL_Net
- Track connection state (Connecting, Connected, ResolutionFailed, ConnectFailed)
- Allocate/deallocate connection slots from a fixed-size pool
- Defer cleanup of completed connections until next allocation request
- Destroy threads safely in destructors

## External Dependencies
- **SDL:** `SDL_Thread`, `SDL_CreateThread()`, `SDL_WaitThread()`, `SDLNet_ResolveHost()`
- **Custom:** `CommunicationsChannel` (defined elsewhere), `IPaddress` type (from cseries.h)
- **Standard Library:** `std::string`, `std::auto_ptr`, `std::pair`

# Source_Files/Network/ConnectPool.h
## File Purpose
Provides a thread-based connection pool for initiating non-blocking outbound TCP connections. Manages up to 20 concurrent connection attempts via `NonblockingConnect` objects and reuses idle connections through the singleton `ConnectPool`.

## Core Responsibilities
- **NonblockingConnect**: Manage a single asynchronous TCP connection attempt
  - Spawn and monitor a background SDL thread for non-blocking connection
  - Track connection lifecycle (Connecting ΓåÆ Connected/Failed)
  - Support DNS resolution (string address) and direct IP connection
  - Wrap successful connections in `CommunicationsChannel` for messaging
- **ConnectPool**: Manage a pool of reusable connection objects
  - Singleton pattern for application-wide access
  - Allocate and recycle `NonblockingConnect` instances
  - Track in-use vs. available slots (bool flag in pair)
  - Clean up abandoned connections

## External Dependencies
- `"config.h"` ΓÇö Provides `DISABLE_NETWORKING` guard and `HAVE_SDL_NET`.
- `"cseries.h"` ΓÇö Core types and macros (uint16 typedef, etc.).
- `"CommunicationsChannel.h"` ΓÇö Defines `CommunicationsChannel`, `IPaddress` (via `<SDL_net.h>`).
- `<string>`, `<memory>` ΓÇö STL containers and smart pointers.
- `<SDL_thread.h>` ΓÇö SDL threading API (`SDL_Thread`).
- **Defined elsewhere:** `IPaddress`, `TCPsocket` (from SDL_net), `CommunicationsChannel`.

# Source_Files/Network/Metaserver/metaserver_dialogs.cpp
## File Purpose
Implements the UI dialog system for Aleph One's metaserver client, allowing players to browse available games, chat, and join network games. Handles game announcement, player interaction, and chat notifications.

## Core Responsibilities
- **Metaserver connection setup**: Initialize player credentials and connect to metaserver with optional update checking
- **Game announcement**: Register active games with the metaserver for visibility to other players
- **Dialog lifecycle management**: Create, run, and tear down the main metaserver browsing UI
- **Chat and player notifications**: Translate metaserver events (player joins, chat messages, game availability) into UI updates
- **Game/player selection and joining**: Manage user interactions with game lists, player lists, and chat
- **UI state management**: Toggle button activation states based on current selection and game compatibility

## External Dependencies
- **Includes**: network_metaserver.h (MetaserverClient, MetaserverPlayerInfo), metaserver_dialogs.h, network_private.h (GAME_PORT), preferences.h (player/network/environment prefs), alephversion.h (version strings), map.h (_force_unique_teams), SoundManager.h, game_wad.h (level_has_embedded_physics_lua), Update.h, progress.h
- **External symbols**: gMetaserverClient, gMetaserverChatHistory (externs); player_preferences, network_preferences, environment_preferences; Scenario::instance(); PlayInterfaceButtonSound(); level_has_embedded_physics_lua(); Update::instance()

# Source_Files/Network/Metaserver/metaserver_dialogs.h
## File Purpose
Defines UI abstractions and handlers for the metaserver client in Aleph One. Provides the main dialog interface for players to browse games, chat, and join servers via the metaserver.

## Core Responsibilities
- Abstract UI layer for metaserver interaction (platform-independent)
- Announce locally-hosted games to the metaserver
- Bridge metaserver notifications (chat, player lists, game updates) to UI widgets
- Manage game/player selection, chat input, and join operations
- Provide factory method for concrete UI implementations (SDL/Carbon determined at link-time)

## External Dependencies
- **`network_metaserver.h`** ΓÇö `MetaserverClient`, `MetaserverClient::NotificationAdapter`, `MetaserverPlayerInfo`
- **`metaserver_messages.h`** ΓÇö `GameListMessage::GameListEntry`, message types
- **`shared_widgets.h`** ΓÇö `PlayerListWidget`, `GameListWidget`, `EditTextWidget`, `ColorfulChatWidget`, `ButtonWidget`
- **Standard library** ΓÇö `<vector>`, `std::auto_ptr`

# Source_Files/Network/Metaserver/metaserver_messages.cpp
## File Purpose
Implements serialization and deserialization (deflation/inflation) of metaserver protocol messages for Aleph One's networking subsystem. Handles encoding/decoding of player information, game descriptions, room listings, chat, and authentication data for TCP communication with metaserver.

## Core Responsibilities
- Serialize message objects to binary streams for network transmission (deflation)
- Deserialize binary data into message objects (inflation)
- Encode/decode player metadata (colors, status, names, teams, ranks)
- Process game descriptions with scenario info, map checksums, and plugin data
- Manage room listings with address/port information
- Support chat and private messaging serialization
- Convert Lua script filenames to human-readable game type strings
- Define static room name lookups and protocol constants

## External Dependencies
- **AStream.h**: `AIStream`, `AOStream` (typed binary stream I/O with `>>`, `<<` operators)
- **Message.h**: `Message`, `SmallMessageHelper`, `DatalessMessage<T>` base classes
- **SDL_net.h**: `IPaddress`, `TCPsocket` network types
- **Boost**: `algorithm::ends_with()`, `algorithm::to_lower_copy()` string utilities
- **Game headers**: `preferences.h`, `shell.h` (color/player prefs), `map.h` (TICKS_PER_SECOND), `network_dialogs.h`, `TextStrings.h` (game type strings), `Scenario.h` (scenario metadata)
- **Defined elsewhere**: `_get_player_color()`, `player_preferences`, `network_preferences`, `get_player_color()`, `TS_GetCString()`, `Scenario::instance()`

---
**Notes:**
- File is conditionally compiled (`#if !defined(DISABLE_NETWORKING)`)
- Heavy use of static helper functions for stream I/O; no class-level state mutation beyond message members
- Metaserver protocol uses fixed-size padded fields mixed with variable-length null-terminated strings
- Room/game lookups index into static tables; name resolution is ID-based for robustness

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

## External Dependencies

- **Message.h:** Base classes `Message`, `SmallMessageHelper` (templated serialization), `DatalessMessage<T>` (zero-payload messages)
- **AStream.h:** Serialization/deserialization streams `AIStream`, `AOStream` with big-endian/little-endian variants
- **SDL_net.h:** IPaddress struct for room server addresses
- **Scenario.h:** `Scenario::instance()` singleton for game scenario metadata (ID, name, version, compatibility checks)
- **network.h:** `kNetworkSetupProtocolID` constant ("Aleph One WonderNAT V1")
- **Boost.Algorithm:** `to_lower_copy()` for case-insensitive string handling
- **Standard library:** `<string>`, `<vector>` for dynamic data

# Source_Files/Network/Metaserver/network_metaserver.cpp
## File Purpose
Implements `MetaserverClient`, a client for connecting to and interacting with an Aleph One metaserver. Handles TCP authentication, player/game list management, chat routing, game announcements, and message dispatching using a handler-based architecture.

## Core Responsibilities
- Establish and maintain TCP connections to metaserver and room servers
- Perform login with optional password encryption (plaintext or XOR-based)
- Maintain player and game lists with add/delete/refresh semantics
- Route incoming network messages to appropriate handlers via dispatcher
- Process chat commands (`.available`, `.who`, `.ignore`, `.games`) and regular messages
- Announce game creation, player counts, start/reset/delete events
- Implement user ignore list with persistent muting and guest filtering
- Provide `pump()` mechanism for non-blocking message I/O processing
- Track all active client instances statically for global updates

## External Dependencies
- **CommunicationsChannel** (defined elsewhere): TCP socket abstraction with message buffering.
- **MessageInflater, MessageDispatcher, MessageHandler** (defined elsewhere): Serialization and message routing.
- **Message and derived classes** (metaserver_messages.h): Protocol message types.
- **MetaserverPlayerInfo, GameListMessage::GameListEntry, RoomDescription** (metaserver_messages.h): Data structures.
- **boost::algorithm::starts_with**: String prefix matching.
- **std::set, std::vector, std::string, std::auto_ptr**: STL containers.
- **Logging.h**: `logAnomaly1()`, `logAnomaly2()` for diagnostics.
- **network_preferences** (external): Preferences object for mute settings.

# Source_Files/Network/Metaserver/network_metaserver.h
## File Purpose
Provides the client-side API for Aleph One games to connect to and interact with a metaserverΓÇöa central hub managing game lobbies, player lists, chat, and game announcements. Handles connection lifecycle, message routing, and notification callbacks to the UI.

## Core Responsibilities
- Manage connection to metaserver (login, disconnect, reconnection)
- Maintain synchronized lists of rooms, players in room, and active games
- Route incoming messages to handlers (chat, private messages, broadcasts, player/game updates)
- Send game lifecycle events (create, start, reset, delete) and player status changes
- Provide notification callbacks for UI updates via observer pattern
- Manage player ignore lists and targeting (selection state)
- Pump network messages on demand or globally across all instances

## External Dependencies

- **metaserver_messages.h**: Message types (`ChatMessage`, `PrivateMessage`, `PlayerListMessage`, `GameListMessage`, `RoomDescription`, `GameDescription`, `MetaserverPlayerInfo`)
- **Logging.h**: Logging macros (`logAnomaly1`)
- **CommunicationsChannel, MessageInflater, MessageDispatcher, MessageHandler**: Defined elsewhere; manage socket I/O, decompression, and message routing
- **Standard library**: `<stdexcept>`, `<exception>`, `<vector>`, `<map>`, `<memory>` (auto_ptr), `<set>`
- **config.h**: Build configuration (networking enabled/disabled)

# Source_Files/Network/Metaserver/NibsMetaserverClientUi.cpp
## File Purpose
Carbon/macOS-specific UI implementation for the metaserver client. Loads Interface Builder NIB files and instantiates a modal dialog with widgets for players, games, chat, and associated controls. Provides the concrete factory implementation of `MetaserverClientUi`.

## Core Responsibilities
- Load and manage Carbon NIB resources ("Metaserver Client")
- Create and initialize UI control widgets (list views, text entry, buttons)
- Implement the `MetaserverClientUi` abstract interface for macOS
- Establish a periodic polling timer to pump metaserver client events at ~30 Hz
- Manage modal dialog lifecycle (Run/Stop)
- Bridge abstract metaserver logic with Carbon framework UI

## External Dependencies
- `metaserver_dialogs.h`: `MetaserverClientUi` base class, `GlobalMetaserverChatNotificationAdapter`
- `NibsUiHelpers.h`: `AutoNibReference`, `AutoNibWindow`, `Modal_Dialog`, `AutoTimer`, `GetCtrlFromWindow()`
- `shared_widgets.h`: `ListWidget`, `ButtonWidget`, `EditTextWidget`, `HistoricTextboxWidget`, `TextboxWidget`
- `boost/function.hpp`, `boost/bind.hpp`, `boost/static_assert.hpp`: Utility templates (minimal use in this file)
- Carbon framework: `EventLoopTimerRef`, `pascal` calling convention (defined elsewhere)
- `MetaserverClient` (singleton, defined elsewhere)
- `GameListMessage::GameListEntry`, `MetaserverPlayerInfo` types (defined elsewhere)

# Source_Files/Network/Metaserver/SdlMetaserverClientUi.cpp
## File Purpose
Implements an SDL-based UI for browsing and joining network games on the Aleph One metaserver. Provides a dialog showing available games, players in rooms, and chat messaging.

## Core Responsibilities
- Constructs and manages the metaserver browser dialog layout (games list, players list, chat)
- Handles periodic updates to game/player lists via a pump function
- Detects and responds to connection loss to the metaserver
- Displays detailed game information on request
- Facilitates game selection and join requests via callbacks

## External Dependencies
- `sdl_dialogs.h`, `sdl_widgets.h`, `sdl_fonts.h` ΓÇö SDL UI framework
- `network_metaserver.h` ΓÇö metaserver client API
- `network_dialog_widgets_sdl.h` ΓÇö `w_games_in_room`, `w_players_in_room` widget definitions
- `interface.h` ΓÇö `set_drawing_clip_rectangle()` for text clipping
- `metaserver_dialogs.h` ΓÇö likely dialog utilities (e.g., `dialog_ok`, `dialog_cancel`, `alert_user`)
- `TextStrings.h` ΓÇö string set lookups (e.g., `TS_GetCString`)
- Boost ΓÇö `boost::function`, `boost::bind` for callback management
- SDL ΓÇö timer (`SDL_GetTicks()`), core event/surface types

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

## External Dependencies
- **Includes:** `"config.h"`, `"cseries.h"`, `"cstypes.h"` (platform abstractions, SDL).
- **Imported:** `struct entry_point` (from map.h), `struct player_start_data` (defined elsewhere), `struct SSLP_ServiceInstance` (service discovery).
- **Callbacks:** Functions like `NetDistributionProc`, `CheckPlayerProcPtr` defined as typedefs; user code registers these.
- **Protocols:** Compile-time constant `kNetworkSetupProtocolID = "Aleph One WonderNAT V1"`; `MARATHON_NETWORK_VERSION` replaced by runtime `get_network_version()` (May 2003 change, implementation in network.c).

# Source_Files/Network/network_audio_shared.h
## File Purpose
Header-only definitions for network audio encoding shared between microphone and speaker modules in Aleph One. Defines the in-memory audio packet header structure and fixed audio format constants used for network transmission.

## Core Responsibilities
- Define `network_audio_header` struct for transmitted audio data
- Define audio format flags (teammate-only filtering)
- Specify network audio format constants (sample rate, bit depth, mono/stereo)
- Provide extensibility hooks (`mReserved` field) for future format changes

## External Dependencies
- `config.h` ΓÇö build configuration (provides `DISABLE_NETWORKING` guard)
- `cseries.h` ΓÇö base types (`uint32`, `int`) and SDL compatibility layer
- Conditional compilation: entire file disabled if `DISABLE_NETWORKING` is defined

# Source_Files/Network/network_capabilities.cpp
## File Purpose
Defines static string constants for network capability flags used in multiplayer protocol negotiation. These constants represent feature identifiers that gatherers and joiners exchange to negotiate compatible protocol versions and optional features (Lua, Speex, zipped data, etc.).

## Core Responsibilities
- Initializes static const string constants that serve as capability identifiers
- Provides a centralized registry of capability flag names for network versioning
- Conditionally compiled when networking is enabled (guarded by `DISABLE_NETWORKING`)
- Part of a capability-negotiation system for game synchronization and optional features

## External Dependencies
- `config.h` ΓÇö build-time configuration (defines `DISABLE_NETWORKING`)
- `network_capabilities.h` ΓÇö forward declares `Capabilities` class and string type aliases
- `<string>` (C++ stdlib) ΓÇö for `std::string` type

**Notes:**
- File is wrapped in `#if !defined(DISABLE_NETWORKING)` guard; entire file is dead code if networking is disabled.
- Version constants (e.g., `kGameworldVersion = 1`) are defined in the header; this file only defines the corresponding string key names.
- The `Capabilities` class uses `map<string, uint32>` to pair these string keys with version numbers at runtime.

# Source_Files/Network/network_capabilities.h
## File Purpose
Defines a versioning and capability declaration system for Aleph One's network layer, allowing gatherers (server) and joiners (clients) to advertise and negotiate supported protocols and data formats. Versions track PRNG/physics, network protocols, scripting, audio streaming, and data compression.

## Core Responsibilities
- Define version constants for network components (gameworld, star/ring protocols, Lua, Speex, etc.)
- Provide a `Capabilities` class that acts as a type-safe map of capability name ΓåÆ version number
- Store static string constants identifying each capability feature
- Enforce bounds checking on capability key size (max 1024 chars)

## External Dependencies
- `<map>`, `<string>` ΓÇô STL containers and strings
- `cseries.h` ΓÇô platform/compiler compatibility layer
- `config.h` ΓÇô build configuration (disable networking flag)

# Source_Files/Network/network_data_formats.cpp
## File Purpose
Implements bidirectional conversion between packed network data formats (wire format) and unpacked in-memory structures for the Aleph One game engine. Ensures consistent byte ordering and padding across all platforms (little-endian and big-endian systems) during network transmission and reception.

## Core Responsibilities
- Convert network packet headers between wire and native formats
- Serialize/deserialize action packets with endianness handling
- Pack/unpack distribution packets for network transmission
- Convert IP address structures (preserving network byte order)
- Handle network audio header serialization
- Maintain byte-order consistency via ValueToStream/StreamToValue utilities
- Provide overloaded `netcpy()` functions for type-safe format conversions

## External Dependencies
- **config.h**: Provides build configuration defines
- **network_data_formats.h**: Struct definitions and extern function declarations
- **Packing.h**: ValueToStream, StreamToValue, ListToStream macros (endianness aware)
- **ALEPHONE_LITTLE_ENDIAN**: Preprocessor define controlling conditional uint32 array conversion
- **DISABLE_NETWORKING**: Guard disabling entire file if networking is disabled

# Source_Files/Network/network_data_formats.h
## File Purpose
Defines cross-platform wire-format structures and conversion functions for network packet serialization. Ensures consistent padding, byte ordering, and alignment across all platforms by providing paired structures (`_NET` for wire format and unpacked for platform format) with conversion utilities (`netcpy` overloads).

## Core Responsibilities
- Define `_NET` structures as binary-compatible wire formats for all network data types
- Provide bidirectional `netcpy()` conversion functions between platform and network formats
- Handle endianness conversion (byte-swapping) based on host architecture
- Define fixed-size constants (SIZEOF_*) for each wire structure
- Support cross-platform game networking without platform-specific code in consumers

## External Dependencies
- `config.h` ΓÇö DISABLE_NETWORKING guard; version/feature flags
- `cseries.h` ΓÇö endianness constant ALEPHONE_LITTLE_ENDIAN; integer types (uint8, uint32, int16, int32)
- `network.h` ΓÇö game_info, player_info type definitions (used by unpacked structures defined elsewhere)
- `network_private.h` ΓÇö NetPacketHeader, NetPacket, NetDistributionPacket (unpacked struct definitions)
- `network_audio_shared.h` ΓÇö network_audio_header (unpacked struct)
- C standard library ΓÇö memcpy() (called inline on big-endian)

# Source_Files/Network/network_ddp.cpp
## File Purpose

Implements DDP (Datagram Delivery Protocol) socket management for Classic Mac OS AppleTalk networking. Provides functions to open/close the AppleTalk driver, create/destroy sockets with packet handlers, allocate/release frames, and asynchronously send DDP packets to remote nodes.

## Core Responsibilities

- Driver lifecycle: open/close the .MPP (MacPack) AppleTalk driver
- Socket lifecycle: open DDP sockets with registered packet handlers, close sockets and clean up buffers
- Frame management: allocate and deallocate DDP frame structures
- Packet transmission: asynchronously send DDP frames with configurable protocol type and destination
- Listener integration: coordinate between C packet handler callbacks and 68k assembly socket listeners via UPPs (Universal Procedure Pointers)

## External Dependencies

- **Apple MacOS APIs:** `OpenDriver()`, `GetResource()`, `HLock()`, `HNoPurge()`, `StripAddress()`, `NewPtrClear()`, `MemError()`, `DisposePtr()`, `NewRoutineDescriptor()`, `CallUniversalProc()`, `GetCurrentISA()`.
- **AppleTalk/MPP macros/functions:** `POpenSkt()`, `PCloseSkt()`, `PWriteDDP()`, `BuildDDPwds()`.
- **Defined elsewhere:** `mppRefNum` (global), socket listener code resource (`SOCK`, ID 128), `DDPSocketListenerUPP` type.

# Source_Files/Network/network_dialog_widgets_sdl.cpp
## File Purpose
Implements SDL-based dialog widgets for network features in Aleph One: player discovery/listing, in-game player display with postgame carnage reports, and level entry-point selection. These widgets integrate with the game's networking subsystem and HUD rendering.

## Core Responsibilities
- `w_found_players`: Widget for listing discovered network players; manages found/hidden/listed player states and callbacks
- `w_players_in_game2`: Dual-purpose widget for displaying active players during gather/join AND as postgame carnage-report graph
- `w_entry_point_selector`: Button widget for choosing level entry points from current map filtered by game type
- Player display: Renders player icons (3D images), names with overlap-avoidance, team/color information
- Carnage statistics: Draws score or kill/death bars, totals, and legends with spatial layout management

## External Dependencies
- **screen_drawing.h**: `draw_text()`, `text_width()`, `set_drawing_clip_rectangle()`, `get_theme_color()`, `get_theme_font()`, color/font access.
- **sdl_fonts.h**: `font_info` class for text measurement and drawing.
- **interface.h**: `get_dialog_player_color()`, `clear_screen()`, theme utilities.
- **network.h**: `NetGetNumberOfPlayers()`, `NetGetPlayerData()`, prospective joiner info.
- **player.h**: `player_data` struct, `get_player_data()`, player constants.
- **HUDRenderer.h**: `PlayerImage` class for rendering 3D player models.
- **shell.h**, **collection_definition.h**: For shape/collection management.
- **preferences.h**, **screen.h**: Environment and map file access.
- **TextStrings.h**: `TS_GetCString()` for localized UI text.
- **TextLayoutHelper.h**: Text layout helper for overlap avoidance.
- **SDL** (via screen_drawing.h): `SDL_Surface`, `SDL_Rect`, `SDL_FillRect()`, `SDL_MapRGB()`.

# Source_Files/Network/network_dialog_widgets_sdl.h
## File Purpose
Defines three custom SDL widget classes for network-related UI dialogs: discovering network players, displaying in-game player status, and selecting map entry points based on game type. Supports both gather/join dialogs and postgame carnage reports.

## Core Responsibilities
- Manage and display lists of remote players discovered via SSLP network protocol
- Render player icons, names, and kill/score statistics in game lobbies and postgame reports
- Validate and present playable entry points (levels) filtered by selected game type
- Delegate user interactions (clicks, selections) to callback handlers

## External Dependencies
- **sdl_widgets.h** ΓÇô `w_list<T>` template, `w_select_button`, `widget` base class
- **SSLP_API.h** ΓÇô `prospective_joiner_info` struct for network-discovered players
- **player.h** ΓÇô `MAXIMUM_PLAYER_NAME_LENGTH`, team colors, player constants
- **PlayerImage_sdl.h** ΓÇô `PlayerImage` class for rendering player icons
- **network_dialogs.h** ΓÇô `net_rank` struct, graphics callback typedefs
- SDL library ΓÇô `SDL_Surface`, `SDL_Event`
- Defined elsewhere: `TextLayoutHelper`, `bar_info` (forward declared)

**Compile guard:** `#if !defined(DISABLE_NETWORKING)` ΓÇô entire file omitted if networking disabled.

# Source_Files/Network/network_dialogs.cpp
## File Purpose
Implements network game dialogs for the Aleph One engine: hosting (gather), joining, and setup flows. Integrates LAN service discovery (SSLP), metaserver for internet games, and pre-game chat. Uses an abstract factory pattern with SDL-specific implementations.

## Core Responsibilities
- Orchestrate gather/join game flows via `network_gather()` and `network_join()` entry points
- Manage LAN service discovery announcements (`GathererAvailableAnnouncer`) and searches (`JoinerSeekingGathererAnnouncer`)
- Implement dialog base classes (`GatherDialog`, `JoinDialog`, `SetupNetgameDialog`) with SDL concrete subclasses
- Handle player list updates, color/team assignment, and player status callbacks
- Coordinate metaserver advertising and internet game search
- Provide progress dialogs for long-running operations
- Support pre-game chat (LAN, internet, and metaserver modes)

## External Dependencies
- **Network engine:** `NetEnter()`, `NetGather()`, `NetStart()`, `NetGameJoin()`, `NetUpdateJoinState()`, etc. (defined elsewhere)
- **Preferences:** `network_preferences`, `player_preferences`, `serial_preferences` (global structs)
- **Metaserver:** `MetaserverClient`, `GameAvailableMetaserverAnnouncer` (external classes)
- **LAN discovery:** `SSLP_Allow_Service_Discovery()`, `SSLP_Locate_Service_Instances()`, `SSLP_Pump()` (SSLP_API.h)
- **Widgets/Dialog:** `w_title`, `w_toggle`, `w_button`, `dialog`, etc. (widget system, defined elsewhere)
- **Utilities:** `getpstr()`, `pstring_to_string()`, `copy_string_to_cstring()`, `alert_user()` (defined elsewhere)
- **Game:** `reassign_player_colors()`, `MAXIMUM_NUMBER_OF_PLAYERS`, `game_info`, `player_info` (defined in player.h, network.h, etc.)

# Source_Files/Network/network_dialogs.h
## File Purpose
Header file defining dialog classes and structures for network game setup, player gathering, joining, and postgame reporting in the Aleph One multiplayer system. Provides abstract base classes with platform-specific implementations (Mac/SDL) for pre-game and postgame UI.

## Core Responsibilities
- Define dialog classes for gathering players (server), joining games (client), and configuring netgame parameters
- Declare structures for player rankings, game outcome data, and network-specific UI state
- Define callback interfaces for network events (player joins/drops, chat messages)
- Declare postgame carnage report visualization functions and data structures
- Provide string IDs and dialog control IDs for resource management across platforms
- Declare helper functions for player color assignment, level selection mapping, and graph rendering

## External Dependencies
- **Headers included**: player.h (MAXIMUM_NUMBER_OF_PLAYERS=8), network.h, network_private.h (JoinerSeekingGathererAnnouncer), FileHandler.h (file dialogs), network_metaserver.h, metaserver_dialogs.h (GlobalMetaserverChatNotificationAdapter), shared_widgets.h (ButtonWidget, EditTextWidget, etc.)
- **Defined elsewhere**: GatherCallbacks, ChatCallbacks (network.h), player_info, game_info (network.h), entry_point (map.h), prospective_joiner_info (network.h), ColorfulChatWidget, FileChooserWidget, EditNumberWidget, SelectorWidget (shared_widgets.h)
- **Conditional compilation**: USES_NIBS (Mac resource forks), DISABLE_NETWORKING (disable all networking)

# Source_Files/Network/network_distribution_types.h
## File Purpose
Centralized header defining enumerated distribution type IDs for network audio in the Aleph One engine. Provides constants used by the network distribution system to identify different styles of audio packet serialization, supporting both legacy and modern network protocols.

## Core Responsibilities
- Define distribution type identifiers for network audio (`kOriginalNetworkAudioDistributionTypeID`, `kNewNetworkAudioDistributionTypeID`)
- Ensure backward compatibility by distinguishing original vs. new-style network audio formats
- Centralize constant definitions to prevent ID conflicts across the networking subsystem
- Gate definitions behind `DISABLE_NETWORKING` compile-time flag for conditional compilation

## External Dependencies
- **Includes:** `config.h` (build configuration flags)
- **Referenced symbols:** `DISABLE_NETWORKING` macro (conditionally disables entire networking subsystem)
- **Used by:** `NetDistributeInformation()`, `NetAddDistributionFunction()` (defined elsewhere; use these enum values to register/identify audio distribution handlers)

# Source_Files/Network/network_dummy.cpp
## File Purpose
Provides stub implementations of network functions when networking is disabled or unavailable. Allows single-player or LAN-disabled builds of the Aleph One game engine to link successfully by providing no-op network interface functions that return safe defaults.

## Core Responsibilities
- Implement all declared network functions with safe dummy behavior
- Enable single-player-only game builds without conditional compilation in game logic
- Return sensible defaults (false, NULL, 0, 1) indicating no network is active
- Disable network-only cheats (crosshair, tunnel vision, behind-view)
- Support map initialization and game queries in non-networked mode

## External Dependencies
- **Includes:**  
  - `cseries.h` ΓÇö core game framework types/macros (includes `cstypes.h` for fixed-width integer types like `int32`, `short`).
  - `map.h` ΓÇö world geometry structures; provides `struct entry_point` for level/spawn data.
  - `network.h`, `network_games.h` ΓÇö function declarations that are implemented here.
- **Defined elsewhere:**  
  - All function signatures are declared in `network.h` and `network_games.h`; this file provides their definitions for DISABLE_NETWORKING builds.
  - `struct entry_point` defined in `map.h`.

# Source_Files/Network/network_games.cpp
## File Purpose

Implements network multiplayer game-mode mechanics for the Aleph One engine (Marathon-compatible). Manages game-type-specific rules, scoring, and state for modes like King of the Hill, Capture the Flag, Rugby, Tag, and Defense. Provides ranking calculations, compass navigation beacons, and end-condition detection.

## Core Responsibilities

- Calculate player and team rankings based on game type and performance metrics
- Initialize game-specific state (beacon locations, ball spawning)
- Provide per-tick game updates (scoring, possession tracking, state transitions)
- Determine end-of-game conditions (kill limits, time limits)
- Track game-specific parameters (flag pulls, hill time, points scored, possession time)
- Generate formatted ranking text for HUD display (scores, percentages, time durations)
- Direct network compass beacons toward objectives (ball, it-player, hill)
- Handle ball possession and destruction in ball-based modes

## External Dependencies

- **map.h:** `polygon_data`, `GET_GAME_TYPE()`, `GET_GAME_OPTIONS()` macros, `dynamic_world`, `map_polygons`, game type enums, `TICKS_PER_SECOND`
- **player.h:** `player_data`, `get_player_data()`, `PLAYER_IS_DEAD()` macro, team color enums
- **items.h:** `find_player_ball_color()`, `BALL_ITEM_BASE`
- **network.h:** Network game declarations (interface only)
- **lua_script.h:** `GetLuaScoringMode()`, `GetLuaGameEndCondition()` (conditional: `HAVE_LUA`)
- **game_window.h:** `mark_player_network_stats_as_dirty()`
- **SoundManager.h:** Sound definitions (not actively used in this file)
- **weapons.c/external:** `destroy_players_ball()` (declaration; definition in weapons.c)

**Defined Elsewhere:** `arctangent()`, `csprintf()`, `vhalt()`, `getcstr()`, `NORMALIZE_ANGLE()`, `QUARTER_CIRCLE`, `HALF_CIRCLE`, `FULL_CIRCLE`, macros for object/player access.

# Source_Files/Network/network_games.h
## File Purpose
Header for network game management in Aleph One (Marathon engine). Declares functions for initializing networked multiplayer games, tracking player/team rankings and scores, generating UI text for scoreboards, and managing network-specific features like compass beacons.

## Core Responsibilities
- Initialize and update the state of network multiplayer games each frame
- Calculate and retrieve player and team rankings with kill/death statistics
- Format ranking and score information for in-game HUD and post-game screens
- Track direct player-on-player kills
- Determine game-over conditions and provide exit entry point flags
- Manage the network compass (beacon) system for player navigation
- Detect game-mode-specific features (scoring, ball/flag modes)

## External Dependencies
- `config.h` ΓÇö feature detection (e.g., `HAVE_SDL_NET`, `DISABLE_NETWORKING` guard)
- Undefined constant `NUMBER_OF_TEAM_COLORS` (likely from game constants header)
- Other network/player/game state headers (not included here)

# Source_Files/Network/network_lookup.cpp
## File Purpose
Implements asynchronous network entity discovery on classic Macintosh AppleTalk using NBP (Name Binding Protocol). Maintains a persistent, alphabetically-sorted list of discovered game servers/clients with automatic stale-entry timeout, optional per-entity filtering, and caller-supplied change notifications.

## Core Responsibilities
- Initiate and manage asynchronous NBP entity lookups (NetLookupOpen/Close)
- Maintain sorted in-memory cache of discovered entities with persistence tracking
- Extract and insert new entities from NBP results; detect duplicates by network address
- Age out stale entities (not heard from in ~4 seconds); notify caller on removal
- Provide filtering capability via caller-supplied callback (e.g., exclude in-game players)
- Notify caller when entities are added/removed via updateProc callback
- Query and enumerate available AppleTalk zones; identify local zone
- Populate macOS menu UI with zone list

## External Dependencies
- **AppleTalk.h**: NBP (PLookupName, PKillNBP, NBPExtract, NBPSetEntity), ZIP (PBControl), XPP (param blocks)
- **macintosh_network.h**: lookupUpdateProcPtr, lookupFilterProcPtr, NetEntityName, AddrBlock, EntityName types
- **macintosh_cseries.h**: Memory (NewPtrClear, DisposePtr, MemError), assertion, string utilities (psprintf, pstrcpy, IUCompString)
- **stdlib.h**: qsort
- **string.h**: Implicit via pstrcpy
- Macintosh Toolbox: SetCursor, GetCursor, CountMenuItems, DeleteMenuItem, AppendMenu, SetMenuItemText, CheckItem, BlockMove
- Conditional: TEST_MODEM flag substitutes ModemLookup* stubs; DISABLE_NETWORKING disables entire file

# Source_Files/Network/network_lookup_sdl.cpp
## File Purpose
Implements network service discovery and player registration for the Aleph One game engine using SDL and SSLP (Simple Service Location Protocol). Handles locating remote player services on the network and publishing this machine's player service for discovery by others.

## Core Responsibilities
- Start/stop service discovery via SSLP callbacks
- Convert game engine's Pascal strings to SSLP-compatible C string format
- Construct and publish local player service instances
- Manage service registration lifecycle (register/unregister)
- Support service discovery hints for cross-broadcast-domain connectivity
- Provide stub functions for entity lookup operations (not yet implemented)

## External Dependencies
- **SDL_net:** `SDLNet_ResolveHost` (resolve address string for hinting)
- **SSLP_API.h:** `SSLP_Locate_Service_Instances`, `SSLP_Stop_Locating_Service_Instances`, `SSLP_Allow_Service_Discovery`, `SSLP_Disallow_Service_Discovery`, `SSLP_Hint_Service_Discovery`
- **cseries.h:** OSErr type, noErr constant, string functions (memcpy, sprintf, strlen, strdup, free)
- **sdl_network.h:** NetAddrBlock (IPaddress alias), NetEntityName type

# Source_Files/Network/network_lookup_sdl.h
## File Purpose
Header declaring SDL-based network service registration and lookup functions for Aleph One's multiplayer networking. Wraps the SSLP (Simple Service Location Protocol) API to provide simpler callback-driven service discovery and publishing on non-AppleTalk networks.

## Core Responsibilities
- Declare service registration/unregistration for network game discovery
- Provide SDL-dialog-friendly lookup interface with callback notifications (found/lost/name-changed)
- Support SSLP hint mechanism for cross-subnet service visibility
- Conditionally expose networking APIs only when `DISABLE_NETWORKING` is not set

## External Dependencies
- `SSLP_API.h` ΓÇô provides `SSLP_Service_Instance_Status_Changed_Callback` typedef and `SSLP_ServiceInstance` struct
- `config.h` ΓÇô for `DISABLE_NETWORKING` guard
- Conditional on `HAVE_SDL_NET` (from config.h)

# Source_Files/Network/network_messages.cpp
## File Purpose
Implements serialization and deserialization ("deflate" and "inflate") operations for network game messages used in multiplayer setup and communication. Handles encoding/decoding of player info, game topology, chat, stats, and supports zlib compression for large data transfers.

## Core Responsibilities
- Serialize/deserialize player network data (addresses, identifiers, player info)
- Implement deflate/inflate for 10+ message types (HelloMessage, TopologyMessage, CapabilitiesMessage, etc.)
- Support zlib compression for large payloads (maps, physics, Lua scripts)
- Manage string encoding (C-strings and Pascal strings) in network format
- Convert between in-memory structures and network byte-order wire format

## External Dependencies
- **AStream.h** ΓÇö `AIStream`, `AIStreamBE`, `AOStream`, `AOStreamBE` for endian-aware serialization
- **zlib.h** ΓÇö `compress()`, `uncompress()` for compression/decompression
- **network_messages.h** ΓÇö Message class declarations (SmallMessageHelper base, message types)
- **network_private.h** ΓÇö `NetPlayer`, `NetTopology`, `game_info`, `player_info`, `ClientChatInfo`, `NetworkStats` structures
- **network_data_formats.h** ΓÇö Data format constants and netcpy functions
- **Logging.h** ΓÇö `logWarning1()` for error reporting
- **cseries.h** ΓÇö Platform-specific types and macros (pstring conversion functions `a1_p2cstr`, `a1_c2pstr`)

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

## External Dependencies
- **config.h**: Conditional compilation flag `DISABLE_NETWORKING`
- **cseries.h**: Standard types (int16, uint8, uint32) and macros
- **AStream.h**: Serialization streams (AIStream, AOStream) for binary I/O with endianness support
- **Message.h**: Base class `Message`, `SmallMessageHelper`, `BigChunkOfDataMessage`, `UninflatedMessage`, `SimpleMessage`, `DatalessMessage`
- **SDL_net.h**: SDL networking primitives (linked via config.h)
- **network_capabilities.h**: `Capabilities` class (stringΓåÆversion map)
- **network_private.h**: `NetPlayer`, `NetTopology`, `CommunicationsChannel`, `MessageDispatcher`, `MessageHandler`, `ClientChatInfo`, `prospective_joiner_info` structs/classes; error codes and distribution packet types

# Source_Files/Network/network_microphone.cpp
## File Purpose

Implements network microphone audio capture for Aleph One (Marathon engine). Handles initialization of Mac OS sound-input devices, manages async recording buffers, converts captured audio to network format with optional Speex compression, and provides start/stop control for in-game voice transmission.

## Core Responsibilities

- Detect and initialize sound-input hardware capability via Gestalt
- Open/close Sound Manager recording devices with proper resource cleanup
- Configure device sample rate, channels, and sample size; tolerate partial failures
- Manage async sound recording via SPB (Sound Parameter Block) callbacks
- Continuously capture audio and route to network transmission layer via `copy_and_send_audio_data()`
- Support optional Speex audio compression (ifdef SPEEX)
- Provide state control interface (`set_network_microphone_state()`) to toggle recording on/off
- Handle A5 world register save/restore for proper Mac 68k callback semantics

## External Dependencies

- **Mac OS Sound Manager:** SPBOpenDevice, SPBCloseDevice, SPBGetDeviceInfo, SPBSetDeviceInfo, SPBRecord, SPBStopRecording, NewSICompletionUPP, DisposeSICompletionUPP
- **Gestalt (Mac OS):** gestaltSoundAttr, gestaltHasSoundInputDevice
- **Speex (optional, ifdef SPEEX):** init_speex_encoder(), destroy_speex_encoder()
- **Network protocol layer (network_microphone_shared.h):** announce_microphone_capture_format(), copy_and_send_audio_data(), get_capture_byte_count_per_packet()
- **Utility macros (cseries.h):** obj_clear()
- **Platform abstractions (cseries.h, shell.h, interface.h):** dprintf()

# Source_Files/Network/network_microphone_coreaudio.cpp
## File Purpose
macOS CoreAudio implementation for capturing microphone input and sending audio data over the network. Handles hardware initialization, format configuration, and real-time audio frame buffering for the Aleph One game engine's network microphone feature.

## Core Responsibilities
- Initialize and configure CoreAudio HAL (Hardware Abstraction Layer) for microphone input
- Discover and select the system's default audio input device
- Negotiate audio format (sample rate, channels, bit depth) between device and application
- Implement audio input callback to receive frames from CoreAudio
- Buffer and accumulate audio data, sending when packet thresholds are met
- Manage microphone capture state (start/stop recording)
- Clean up audio unit and allocated memory on shutdown

## External Dependencies
- **Frameworks:** Carbon/Carbon.h, AudioUnit/AudioUnit.h (CoreAudio on macOS).
- **Internal headers:** `network_microphone_shared.h` (declares `announce_microphone_capture_format()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()`); `cstypes.h` (provides `uint8`, `Uint32`).
- **Conditional:** Speex encoder init/destroy functions (defined elsewhere, gated by `#ifdef SPEEX`).
- **Standard library:** `<vector>` for `captureBuffer`, `<cstdio>` implicitly for `fprintf()`.

# Source_Files/Network/network_microphone_sdl_alsa.cpp
## File Purpose
Implements ALSA (Advanced Linux Sound Architecture) microphone audio capture for Aleph One's network voice chat. Configures the audio device, manages async capture callbacks, and pipes captured audio to the network transmission layer with optional Speex compression.

## Core Responsibilities
- Open and configure ALSA PCM capture device with hardware parameters (sample rate, format, channels)
- Configure software parameters (availability threshold, start threshold) for low-latency capture
- Register asynchronous capture callback to process incoming audio data
- Manage microphone state transitions (prepare/start/drop device)
- Integrate optional Speex compression encoder initialization/cleanup
- Provide fallback dummy implementation when ALSA is unavailable

## External Dependencies
- **ALSA library:** `#include <alsa/asoundlib.h>` ΓÇö PCM device I/O and parameter configuration
- **Speex codec (conditional):** `#include "network_speex.h"` ΓÇö optional voice compression; encoder/decoder state globals and init/destroy functions
- **Network layer (shared interface):** `#include "network_microphone_shared.h"` ΓÇö `announce_microphone_capture_format()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()`
- **Preferences:** `#include "preferences.h"` ΓÇö `network_preferences` global for Speex encoder flag
- **Core library:** `#include "cseries.h"` ΓÇö common types and macros
- **Standard I/O:** `fprintf(stderr, ...)` for error logging

# Source_Files/Network/network_microphone_sdl_dummy.cpp
## File Purpose
Dummy/stub implementation of SDL-based network microphone functionality for the Aleph One game engine. Provides no-op implementations to satisfy the linker when actual network microphone support is not available or needed. Explicitly returns `false` for implementation check to declare unsupported state.

## Core Responsibilities
- Provide stub `open_network_microphone()` function for initialization
- Provide stub `close_network_microphone()` function for shutdown
- Provide stub `set_network_microphone_state()` to accept but ignore activation requests
- Provide `is_network_microphone_implemented()` returning `false` to declare lack of support
- Provide stub `network_microphone_idle_proc()` for frame/idle updates

## External Dependencies
- No includes or external symbols visible in this file
- All functions are self-contained stubs with no dependencies

# Source_Files/Network/network_microphone_sdl_win32.cpp
## File Purpose
DirectSoundCapture (Win32 DirectX)-based network microphone implementation for Marathon: Aleph One. Captures audio input from the system microphone via a circular buffer and transmits it over the network for multiplayer voice communication.

## Core Responsibilities
- Initialize and configure DirectSoundCapture device with hardware format detection
- Manage circular audio capture buffer and read position tracking
- Select optimal audio format from device-supported options (11 kHzΓÇô44 kHz, mono/stereo, 8-bit/16-bit)
- Continuously capture and transmit audio data in packet-sized chunks during idle processing
- Start/stop audio transmission on demand
- Integrate with optional Speex audio compression codec
- Clean up DirectX resources on shutdown

## External Dependencies
- `<dsound.h>`: DirectSoundCapture API (Windows DirectX)
- `cseries.h`: Engine primitives and types
- `network_microphone_shared.h`: Shared interface functions (`announce_microphone_capture_format`, `copy_and_send_audio_data`, `get_capture_byte_count_per_packet`)
- `network_speaker_sdl.h`: Audio definitions
- `Logging.h`: Logging macros (`logContext`, `logAnomaly`, `logAnomaly1`, `logAnomaly3`)
- `preferences.h`: Global `network_preferences` (Speex encoder flag)
- `network_speex.h`: Optional codec (`init_speex_encoder`, `destroy_speex_encoder`) ΓÇô conditional on `SPEEX` macro

# Source_Files/Network/network_microphone_shared.cpp
## File Purpose

Provides platform-independent utility routines for network microphone audio capture, resampling, and transmission in Marathon: Aleph One's multiplayer system. Handles audio format conversion (mono/stereo, 8/16-bit), rate resampling to network standard, and Speex compression for efficient network distribution.

## Core Responsibilities

- Announce and store microphone capture format (sample rate, stereo/mono, bit depth)
- Calculate raw capture buffer sizes per network packet based on format parameters
- Extract audio samples from raw capture buffers with format conversions
- Resample audio to network standard sample rate using fixed-point arithmetic
- Encode resampled audio frames using Speex compression
- Manage two-chunk circular buffer input for capture callbacks
- Distribute encoded packets to network or loopback handler
- Support debug-mode local loopback for testing

## External Dependencies

- **network_data_formats.h** ΓÇö `network_audio_header`, `netcpy()` (endian-aware copy)
- **network_distribution_types.h** ΓÇö `kNewNetworkAudioDistributionTypeID`
- **network_speaker_sdl.h** ΓÇö Speaker interface (headers only, no direct calls here)
- **network_speex.h** ΓÇö `gEncoderState`, `gEncoderBits`, Speex state (conditional SPEEX)
- **preferences.h**, **map.h** ΓÇö `GET_GAME_OPTIONS()`, `_force_unique_teams` flag
- **cseries.h** ΓÇö Platform types, `_fixed`, `FIXED_ONE`, STL
- **algorithm** ΓÇö `std::min`, `std::pair`
- **speex/speex.h** ΓÇö Speex codec (conditional SPEEX): `speex_bits_reset`, `speex_encode_int`, `speex_bits_write`

**External symbols called but not defined here:**
- `NetDistributeInformation()` ΓÇö Network packet distribution (defined elsewhere)
- `received_network_audio_proc()` ΓÇö Debug loopback handler (conditional, defined elsewhere)
- `kNetworkAudioSampleRate`, `kNetworkAudioBytesPerFrame` ΓÇö Constants (likely in network_audio_shared.h)

**Compile guards:** Entire file wrapped in `!DISABLE_NETWORKING`; SPEEX codec sections conditional on `SPEEX` define.

# Source_Files/Network/network_microphone_shared.h
## File Purpose
Defines the internal interface for network microphone (netmic) implementations to announce audio capture format, transmit captured audio data packets over the network, and query packet size requirements. Implementation-specific; not intended for general engine code.

## Core Responsibilities
- Define the contract for audio format announcement (sample rate, stereo/mono, bit depth)
- Provide buffering and packetization logic for captured audio transmission
- Calculate optimal packet size based on current capture format
- Guard implementation against incomplete or mismatched format specifications

## External Dependencies
- `config.h` (conditional compilation guard: `DISABLE_NETWORKING`)
- All implementation details ("defined elsewhere") ΓÇö functions likely defined in sibling netmic implementation files

# Source_Files/Network/network_names.cpp
## File Purpose
Manages registration and unregistration of AppleTalk network entity names for the local game instance. Provides a wrapper around the Mac Toolbox NBP (Name Binding Protocol) to advertise the game's presence on the network with a socket number and type identifier.

## Core Responsibilities
- Register a single entity name on the network via AppleTalk NBP
- Store the names table entry in system heap to survive application crashes
- Construct adjusted type names by appending version numbers
- Unregister and deallocate the names table entry on shutdown
- Optionally support modem-based name registration (test mode)

## External Dependencies
- **`#include "macintosh_cseries.h"`** ΓÇô Mac platform utilities (string functions, memory allocation)
- **`#include "macintosh_network.h"`** ΓÇô Game network header (provides function prototypes and AppleTalk type definitions)
- **`#include <AppleTalk.h>`** (via macintosh_network.h) ΓÇô Mac Toolbox AppleTalk definitions (NBP, NamesTableEntry, etc.)
- **Defined elsewhere:** `GetString()`, `pstrcpy()`, `psprintf()`, `NewPtrClear()`, `NewPtrSysClear()`, `DisposePtr()`, `MemError()`, `NBPSetNTE()`, `PRegisterName()`, `PRemoveName()`, `ModemRegisterName()`, `ModemUnRegisterName()`

# Source_Files/Network/network_private.h
## File Purpose
Private header for the network subsystem containing internal data structures, packet formats, and service discovery mechanisms. This file defines the ring protocol packet structures, network topology representation, and error codes used across the networking subsystem but intentionally hidden from game code outside the network module.

## Core Responsibilities
- Define packet headers and formats for the ring protocol (UDP-based game synchronization)
- Define network topology and player structures for multi-player state management
- Define packet type tags, enums, and protocol constants for message routing
- Support distribution of arbitrary data types (lossy/lossless) via the network
- Define service discovery (Zeroconf-like) for gatherer/joiner discovery using SSLP
- Define error codes and chat messaging structures
- Provide constants for timing, buffer sizes, and network parameters

## External Dependencies
- **config.h** ΓÇô build-time configuration flags (e.g., DISABLE_NETWORKING)
- **cstypes.h** ΓÇô platform-specific integer types (int16, int32, uint8, uint32, etc.)
- **sdl_network.h** ΓÇô SDL networking definitions (IPaddress, TCPsocket, UDPsocket, etc.)
- **network.h** ΓÇô public network API and callback interfaces (game_info, player_info, NetDistributionProc, etc.)
- **SSLP_API.h** ΓÇô service discovery API (SSLP_ServiceInstance, SSLP_Allow_Service_Discovery, etc.)
- **memory** (C++ standard library) ΓÇô for std::string in ClientChatInfo
- **Defined elsewhere**: NetGetDistributionInfoForType, get_network_version(), various ring-protocol implementations

# Source_Files/Network/network_sound.h
## File Purpose
Main interface header for network audio support in Aleph One (Marathon engine). Declares functions for bidirectional network audio: speaker system for receiving/playing back remote player audio, and microphone system for capturing/transmitting local player audio over the network.

## Core Responsibilities
- Initialize, control, and shut down network speaker (remote audio playback) system
- Queue incoming network audio data for playback and silence the speaker
- Mute individual players' microphones or clear all mutes
- Detect and initialize network microphone (audio capture) capabilities
- Activate/deactivate local microphone and process captured audio for transmission
- Provide idle entry points for periodic audio processing during game loop

## External Dependencies
- **Includes**: `config.h` (configuration constants), `cseries.h` (platform abstractions, types).
- **Defined elsewhere**: 
  - `OSErr` type (platform error code, likely from cseries).
  - `byte` type (likely from cstypes).
  - Implementations: `NETWORK_SPEAKER.C`, `NETWORK_MICROPHONE.C`.

# Source_Files/Network/network_speaker.cpp
## File Purpose
Manages playback of received network audio data for multiplayer games. Handles double-buffered audio output via Mac Sound Manager APIs, maintains a queue of incoming network audio, and implements a state machine to start/stop audio playback based on connection status and data availability.

## Core Responsibilities
- Initialize and tear down network audio playback device (`open_network_speaker`, `close_network_speaker`)
- Queue incoming network audio data from remote players (interrupt-safe)
- Fill double buffers with queued audio or generated static when data unavailable
- Manage speaker state machine (off ΓåÆ turning on ΓåÆ on)
- Monitor connection status and suppress audio if no data arrives within threshold
- Generate pseudo-random static/noise to fill gaps and reduce startup delay
- Coordinate with main thread idle processing to start audio playback safely

## External Dependencies
- **Includes:** `stdlib.h`, `macintosh_cseries.h`, `CarbonSndPlayDB.h` (Carbon only), `network_sound.h`, `network_speex.h` (conditional), `Logging.h`
- **External symbols used (defined elsewhere):**
  - Sound Manager types/functions: `SndChannelPtr`, `SndDoubleBufferPtr`, `SndDoubleBufferHeaderPtr`, `SndNewChannel`, `SndDisposeChannel`, `SndDoImmediate`, `SndPlayDoubleBuffer`, `CarbonSndPlayDoubleBuffer` (Carbon), `MySndDoImmediate` (Carbon)
  - Memory management: `NewPtr`, `NewPtrClear`, `DisposePtr`, `MemError`
  - System: `atexit`, `BlockMove`
  - Logging: `logAnomaly3`, `warn`, `vhalt`, `csprintf`
  - Speex codec (conditional): `init_speex_decoder`, `destroy_speex_decoder`
  - Utilities: `MIN` macro

# Source_Files/Network/network_speaker_sdl.cpp
## File Purpose

Implements real-time network audio playback for SDL platforms in Marathon: Aleph One. Manages queues of sound buffers received from the network, provides padding noise during gaps, and interfaces with the Mixer to feed audio data to the output system. Centralizes memory management to ensure all allocations/deallocations occur on the main thread.

## Core Responsibilities

- Initialize and shut down network speaker resources (buffers, queues, noise data)
- Queue incoming network audio packets for playback
- Manage a circular queue of reusable audio storage buffers
- Provide synthetic noise buffers during playback underruns
- Track consecutive empty dequeues to detect when playback should stop
- Interface with the audio Mixer to provide/retrieve buffer descriptors
- Handle optional Speex codec initialization (if compiled with `SPEEX`)

## External Dependencies

- **Includes:**
  - `network_sound.h` ΓÇö interface declarations (public API)
  - `network_speaker_sdl.h` ΓÇö `NetworkSpeakerSoundBufferDescriptor`, `is_sound_data_disposable()`
  - `CircularQueue.h` ΓÇö queue template implementation
  - `world.h` ΓÇö `local_random()`
  - `Mixer.h` ΓÇö `Mixer::instance()`, audio system integration
  - `network_distribution_types.h` ΓÇö distribution type constants (included but unused in this file)
  - `network_speex.h` ΓÇö `init_speex_decoder()`, `destroy_speex_decoder()` (conditional)

- **Defined elsewhere:**
  - `fdprintf()` ΓÇö diagnostic output (likely in cseries/logging)
  - `Mixer` singleton ΓÇö drives audio mixing and playback
  - `local_random()` ΓÇö pseudo-random number generation

# Source_Files/Network/network_speaker_sdl.h
## File Purpose
Interface between SDL network speaker receiving code and SDL sound playback code. Defines structures and functions for dequeuing incoming network audio (realtime microphone data) and managing buffer lifecycle on SDL platforms.

## Core Responsibilities
- Define the `NetworkSpeakerSoundBufferDescriptor` structure for passing audio data between network and sound subsystems
- Provide flag-based metadata (disposability) for buffer management
- Dequeue incoming network audio data for playback
- Return used buffers to the free queue for reuse

## External Dependencies
- `#include "config.h"` ΓÇö conditional compilation guard (DISABLE_NETWORKING)
- `#include "cseries.h"` ΓÇö provides base types (`byte`, `uint32`), macros, and SDL integration
- Implied: threading/synchronization primitives for queue management (not visible; defined elsewhere)

# Source_Files/Network/network_speaker_shared.cpp
## File Purpose
Handles receiving and decoding networked audio from remote players for local playback. Manages player muting and audio filtering based on team settings. Decodes compressed Speex audio when available.

## Core Responsibilities
- Receive network audio packets and decode compressed frames (Speex)
- Filter audio based on team-only chat settings and local player team
- Queue decoded audio data for playback via the speaker system
- Manage player microphone mute state (per-player ignore list)
- Clear all player mutes

## External Dependencies
- **network_sound.h**: queue_network_speaker_data() (playback queuing), forward declarations
- **network_data_formats.h**: network_audio_header_NET struct, netcpy() (networkΓåöhost format conversion)
- **network_audio_shared.h**: network_audio_header struct, kNetworkAudioForTeammatesOnlyFlag, audio format constants
- **player.h**: player_data struct, get_player_data(), local_player global
- **shell.h**: screen_printf()
- **speex/speex.h** (optional, #ifdef SPEEX): speex_bits_read_from(), speex_decode_int()
- **network_speex.h**: gDecoderBits, gDecoderState globals
- **cseries.h**: Common engine types (byte, uint32, etc.)
- Standard: \<set\>

# Source_Files/Network/network_speex.cpp
## File Purpose
Provides Speex encoder/decoder initialization and resource management for network audio compression in Aleph One. Configures audio preprocessing with automatic gain control (AGC) and denoising for real-time voice communication.

## Core Responsibilities
- Initialize Speex narrowband encoder with fixed quality (3) and complexity (4) settings
- Initialize Speex narrowband decoder with enhancement enabled
- Configure audio preprocessing with denoise and automatic gain control
- Manage lifecycle and cleanup of encoder/decoder state and bitstream objects
- Ensure single initialization via guard conditions

## External Dependencies
- **Speex library** (defined elsewhere):
  - `speex_encoder_init()`, `speex_decoder_init()`, `speex_nb_mode` (narrowband codec mode)
  - `speex_encoder_ctl()`, `speex_decoder_ctl()` (configuration)
  - `speex_bits_init()`, `speex_bits_destroy()` (bitstream management)
  - `speex_preprocess_state_init()`, `speex_preprocess_ctl()` (audio preprocessing)
- **Game constants:** `kNetworkAudioSampleRate` (8000 Hz) from `network_audio_shared.h`
- **Conditional:** Entire file gated by `SPEEX` and `!DISABLE_NETWORKING` preprocessor guards

# Source_Files/Network/network_speex.h
## File Purpose
Header file declaring global Speex encoder/decoder state and management functions for network audio compression in Aleph One. Provides interface for initializing and cleaning up Speex codec resources used in multiplayer audio streaming.

## Core Responsibilities
- Declare global encoder/decoder state pointers and bit buffers
- Declare audio preprocessing state for Speex
- Declare initialization function for Speex encoder
- Declare cleanup function for Speex encoder
- Declare initialization function for Speex decoder
- Declare cleanup function for Speex decoder

## External Dependencies
- **`speex/speex.h`** ΓÇô Speex codec library (encoder/decoder/bits API)
- **`speex/speex_preprocess.h`** ΓÇô Speex preprocessing (noise suppression, AGC)
- **`cseries.h`** ΓÇô Aleph One common utilities and types
- **`config.h`** ΓÇô Build configuration; `DISABLE_NETWORKING` and `SPEEX` checks

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

## External Dependencies
- **TickBasedCircularQueue.h**: `ConcreteTickBasedCircularQueue<T>` and `WritableTickBasedCircularQueue<T>` templates for tick-indexed queues.
- **ActionQueues.h**: Additional queue utilities (included but action_flags_t uses the TickBasedCircularQueue).
- **sdl_network.h**: `DDPPacketBufferPtr`, `NetAddrBlock`, packet I/O definitions.
- **map.h**: `TICKS_PER_SECOND` constant (30 ticks/sec in standalone hub mode).
- **XML_ElementParser** (forward-declared): Used for configuration parsing.
- Defined elsewhere: `kNetLatencyInvalid`, `kNetLatencyDisconnected` (constants), packet magic constants.

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

# Source_Files/Network/network_udp.cpp
## File Purpose
Implements UDP datagram networking for Aleph One, replacing the old AppleTalk DDP protocol. Provides cross-platform UDP socket management via SDL_net, with dedicated thread-based packet reception and a callback-driven packet handler model.

## Core Responsibilities
- Initialize and shutdown the UDP network module
- Open/close UDP sockets and bind to ports
- Spawn and manage a dedicated receiver thread with high priority
- Receive incoming UDP packets and dispatch them via registered packet handlers
- Create/dispose of DDP frame structures for outgoing packets
- Send frame data to remote addresses via UDP

## External Dependencies
- **SDL_net**: `UDPsocket`, `UDPpacket`, `SDLNet_*` functions (socket, packet, and set operations).
- **SDL**: `SDL_Thread`, `SDL_CreateThread()`, `SDL_WaitThread()`.
- **mytm.h**: `take_mytm_mutex()`, `release_mytm_mutex()` for thread-safe handler dispatch.
- **thread_priority_sdl.h**: `BoostThreadPriority()` for elevating receiver thread priority.
- **Defined elsewhere:** `PacketHandlerProcPtr` callback type, `DDPPacketBuffer` / `DDPFrame` structures, `NetAddrBlock` address type, `kPROTOCOL_TYPE` constant, `fdprintf()` logging function.

# Source_Files/Network/network_udp_opentransport.cpp
## File Purpose
Implements UDP networking for classic Mac OS via Apple's OpenTransport API, providing the DDP (Datagram Delivery Protocol) abstraction layer. Acts as a bridge between the game's network protocol (ring-based) and low-level UDP transport, using asynchronous notifications to handle incoming packets and a deferred-send queue to avoid interrupt-time operations.

## Core Responsibilities
- Initialize and teardown OpenTransport framework (`NetDDPOpen`, `NetDDPClose`)
- Create, bind, and configure UDP socket endpoints (`NetDDPOpenSocket`, `NetDDPCloseSocket`)
- Receive incoming UDP datagrams via asynchronous OpenTransport notifications
- Queue outgoing UDP frames for deferred transmission (`NetDDPSendFrame`, `NetDDPSendUnsentFrames`)
- Manage packet allocation and deallocation (`NetDDPNewFrame`, `NetDDPDisposeFrame`)
- Forward received packets to caller-supplied handler via callback
- Handle OpenTransport event notifications (data arrival, flow control, errors)

## External Dependencies
- **OpenTransport API:** `OTCreateConfiguration`, `OTOpenEndpoint`, `OTBind`, `OTSetBlocking`, `OTSetAsynchronous`, `OTInstallNotifier`, `OTRcvUData`, `OTSndUData`, `OTRcvUDErr`, `OTRemoveNotifier`, `OTUnbind`, `OTCloseProvider`, `OTSetSynchronous`, `OTSetNonBlocking`, `OTEnterNotifier`, `OTLeaveNotifier`, `NewOTNotifyUPP`, `DisposeOTNotifyUPP`, `InitOpenTransport` (Classic Mac OS only; guarded by `#ifndef __MACH__`)
- **SDL_net:** via `sdl_network.h` (comment notes "you don't even want to ask" why included)
- **SSLP:** via `network_private.h` (service location)
- **Logging:** `logError1()`, `logNote()`, `logAnomaly1()`, `logTrace2()` from `Logging.h`
- **Network internals:** `kPROTOCOL_TYPE`, `ddpMaxData` from `network_private.h` and `sdl_network.h`
- **Utilities:** `obj_clear()` (from cseries.h), `memcpy()` (standard C)

# Source_Files/Network/NetworkGameProtocol.cpp
## File Purpose
This is a conditional compilation wrapper for the network game protocol module. It includes the NetworkGameProtocol header only when networking is enabled (DISABLE_NETWORKING is not defined). The file itself contains no implementation.

## Core Responsibilities
- Conditionally include the NetworkGameProtocol header based on build configuration
- Guard against compilation when networking is disabled
- Serve as the primary translation unit for network protocol definitions

## External Dependencies
- **config.h** ΓÇö Build-time configuration header; provides `DISABLE_NETWORKING` macro and version information
- **NetworkGameProtocol.h** ΓÇö Abstract base class interface defining the game protocol contract; conditionally included only when networking is enabled

---

**Note:** The actual implementation is essentially empty. The abstract interface (`NetworkGameProtocol` class) is defined in the header. This .cpp file is likely a placeholder, with concrete protocol implementations in other files within the Network subsystem.

# Source_Files/Network/NetworkGameProtocol.h
## File Purpose
Abstract base class defining the interface for game protocol implementations in the Aleph One network subsystem. Establishes contracts for synchronization, information distribution, packet handling, and action flag management across network participants.

## Core Responsibilities
- Define the contract for network protocol implementations (pure virtual interface)
- Manage session lifecycle (initialization, synchronization, graceful shutdown)
- Distribute game information to network participants
- Handle incoming network packets
- Coordinate action flag synchronization and prediction for gameplay
- Provide network timing for game tick synchronization

## External Dependencies
- `network_private.h` ΓÇö Provides `NetTopology`, `DDPPacketBuffer` structures and network constants
- `config.h` ΓÇö Compile-time configuration (DISABLE_NETWORKING guard)

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

## External Dependencies
- **Includes:** `config.h`, `cseries.h`, `Carbon.h` (Mac), `ActionQueues.h`, `player.h`, `network.h`, `network_private.h`, `network_data_formats.h`, `mytm.h`, `map.h`, `vbl.h`, `interface.h`, `XML_ElementParser.h`, `Logging.h`
- **External symbols:** `GetRealActionQueues()`, `process_action_flags()` (interface.h), `NetDDPNewFrame()`, `NetDDPDisposeFrame()`, `NetDDPSendFrame()` (sdl_network.h), `myTMRemove()`, `myXTMSetup()` (mytm.h), `alert_user()`, `dynamic_world` (game state), `machine_tick_count()` (timing), `netcpy()` (network_data_formats.h), `logNote1()`, `logDump2()` (Logging.h)
- **Conditionally enabled:** Multiple adaptive latency schemes; debug statistics and streaming (STREAM_NET); modem sync fallback.

# Source_Files/Network/RingGameProtocol.h
## File Purpose
Defines `RingGameProtocol`, a concrete implementation of the `NetworkGameProtocol` interface for the "Aleph One" game engine's ring-based network protocol. Provides the interface between the legacy ring protocol module and the rest of the game, enabling multiplayer state synchronization and packet handling.

## Core Responsibilities
- Initialize and teardown network protocol state (two-phase Enter/Exit lifecycle)
- Distribute game information across the network to all players or teams
- Synchronize and desynchronize game state with network topology
- Handle incoming DDP (network) packets
- Manage network time for game tick alignment
- Track and predict unconfirmed player action flags for client-side prediction
- Parse XML-based configuration preferences for ring protocol settings

## External Dependencies
- **Inheritance**: `NetworkGameProtocol` (abstract base, defined in NetworkGameProtocol.h)
- **Forward declarations**: `XML_ElementParser`, `DDPPacketBuffer`, `NetTopology` (defined elsewhere)
- **Includes**: `<stdio.h>` (FILE type), `config.h` (preprocessor configuration)
- **Types used but not defined**: `int32`, `uint32`, `size_t`, `short` (likely from stdint.h or platform headers)

# Source_Files/Network/SDL_netx.cpp
## File Purpose
Extends SDL_net with UDP broadcast capabilities not natively supported by the library. Provides platform-abstracted functions to enable broadcast mode on sockets and send packets to all available network broadcast addresses. Handles platform-specific variations (Windows, BeOS, Mac, Linux/BSD).

## Core Responsibilities
- Enable/disable the SO_BROADCAST socket option on UDP sockets
- Enumerate network interfaces and their broadcast addresses (Unix/BSD platforms)
- Send UDP packets to all enumerated broadcast addresses or a platform-appropriate fallback
- Abstract platform differences (Windows accepts 255.255.255.255; Unix requires explicit interface enumeration)
- Manage cached broadcast address list to avoid repeated system calls

## External Dependencies
- **SDL_net:** `UDPsocket`, `UDPpacket`, `SDLNet_UDP_Send()`
- **System socket APIs:** `setsockopt()`, `ioctl()`, `socket.h`, `netinet/in.h`, `net/if.h`
- **Platform-specific headers:** `winsock.h` (Windows), system headers vary by OS
- **Compiler/platform conditionals:** `WIN32`, `__BEOS__`, `mac`, `__MWERKS__`, `__svr4__`

# Source_Files/Network/SDL_netx.h
## File Purpose
Header file providing UDP broadcast capabilities for SDL_net. Extends SDL_net with functions to enable broadcast sending across all available network interfaces, addressing functionality that base SDL_net lacks. Part of the Aleph One game engine networking subsystem.

## Core Responsibilities
- Enable/disable SO_BROADCAST socket option on UDP sockets
- Send UDP packets to broadcast addresses across all network interfaces
- Manage broadcast state and validate socket readiness

## External Dependencies
- `SDL_net.h` ΓÇö provides `UDPsocket`, `UDPpacket` types
- `config.h` ΓÇö guards code with `DISABLE_NETWORKING` preprocessor flag; requires `HAVE_SDL_NET`

# Source_Files/Network/SSLP_API.h
## File Purpose
Public API header for SSLP (Simple Service Location Protocol)ΓÇöa custom service discovery mechanism for finding game servers on networks. Designed for the Aleph One project to enable player-to-server discovery without relying on AppleTalk. Intentionally simplified for game server discovery use cases rather than heavyweight enterprise deployment.

## Core Responsibilities
- Define the `SSLP_ServiceInstance` data structure for advertising services
- Provide service location/discovery APIs for clients seeking remote services
- Provide service publishing APIs for servers advertising themselves
- Support callback-based notification of service state changes (found, lost, name changed)
- Abstract over single-threaded and multi-threaded implementations
- Integrate with SDL_net for cross-platform UDP-based communication

## External Dependencies
- `SDL_net.h` ΓÇö Cross-platform UDP/IP networking primitives (`IPaddress` type)
- `config.h` ΓÇö Build-time configuration flags; guards entire header with `!DISABLE_NETWORKING`

**Notes**: Callbacks may execute in arbitrary threads if threading is enabled. Service instance pointers are valid only during their advertised lifetime (until lost callback or explicit stop). No guarantee of service existence at advertised addressΓÇöSSLP is best-effort hints only.

# Source_Files/Network/SSLP_limited.cpp
## File Purpose
Implementation of the Simple Service Location Protocol (SSLP) for network service discovery. Enables applications to locate remote services (FIND), advertise local services (HAVE), and notify when services become unavailable (LOST). Single-threaded design requires periodic calls to SSLP_Pump() from the main thread to process network activity.

## Core Responsibilities
- Manage UDP socket for SSLP packet transmission and reception
- Maintain linked list of discovered service instances with timeout tracking
- Handle three message types: FIND (requests), HAVE (responses), LOST (notifications)
- Track active behaviors via state flags (LOCATING, RESPONDING, HINTING)
- Serialize/deserialize packets with architecture-independent packing
- Invoke user-provided callbacks when services are found, lost, or renamed
- Broadcast FIND requests and listen for responses at regular 5-second intervals

## External Dependencies
- **SDL.h, SDL_endian.h:** SDL type definitions (Uint32, Uint16, SDL_GetTicks, SDL_SwapBE32/16)
- **SDL_net.h:** UDP socket/packet APIs (SDLNet_UDP_Open/Close, Alloc/FreePacket, Recv/Send, UDPsocket, UDPpacket, IPaddress)
- **SDL_netx.h:** Broadcast extensions (SDLNetx_EnableBroadcast, SDLNetx_UDP_Broadcast)
- **SSLP_API.h, SSLP_Protocol.h:** Protocol definitions (constants, SSLP_Packet, SSLP_ServiceInstance, callback typedef)
- **Logging.h:** Diagnostic output (logContext, logTrace, logAnomaly, logNote)
- **Standard C:** assert, string (strncmp, strncpy, memcpy, memset), stdlib (malloc, free)

# Source_Files/Network/SSLP_Protocol.h
## File Purpose
Defines the network protocol specification for SSLP (Simple Service Location Protocol), enabling service discovery on non-AppleTalk networks for Aleph One game servers. Specifies packet format, constants, and protocol semantics for finding and advertising network services via UDP broadcast/unicast.

## Core Responsibilities
- Define SSLP packet structure (`struct SSLP_Packet`) for network wire format
- Establish protocol constants: magic number, version, message types (FIND/HAVE/LOST)
- Configure network port and service type/name field constraints
- Document protocol behavior: broadcast discovery, unicast responses, loss tolerance, unsolicited announcements
- Ensure cross-platform binary compatibility through packed struct layout and big-endian field ordering

## External Dependencies
- **SDL_net.h**: Cross-platform UDP/IP networking library (datagram support required)
- **config.h**: Build configuration; guards entire section with `DISABLE_NETWORKING`


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

## External Dependencies
- **Network**: `network_star.h` ΓÇö hub/spoke implementations (`hub_initialize`, `spoke_initialize`, `hub_received_network_packet`, `spoke_received_network_packet`, etc.)
- **Queue abstraction**: `TickBasedCircularQueue.h` ΓÇö `WritableTickBasedCircularQueue`, `ConcreteTickBasedCircularQueue`
- **Player / Actions**: `player.h` ΓÇö `GetRealActionQueues()` (returns active action queue set)
- **Interface**: `interface.h` ΓÇö `process_action_flags()` (delivers action flags to player update logic)
- **XML**: Not explicitly included; `XML_ElementParser` assumed defined elsewhere
- **Networking**: `cseries.h` (platform types, build config); `DDPPacketBuffer` type (assumed from headers not shown)
- **Distribution**: `NetGetDistributionInfoForType()`, `NetDistributionInfo` (defined elsewhere)

# Source_Files/Network/StarGameProtocol.h
## File Purpose
Defines the interface for a star-topology network game protocol implementation in Aleph One. StarGameProtocol inherits from the abstract NetworkGameProtocol base class and specifies the complete protocol behavior for multi-player games using a star (centralized server) network topology.

## Core Responsibilities
- **Lifecycle management**: Enter/Exit phases for game session establishment and teardown
- **Message distribution**: Broadcast information packets to all players or teams
- **Network synchronization**: Coordinate game state and ticks across the network
- **Packet handling**: Receive and process incoming network packets
- **Action flag prediction**: Track unconfirmed action flags for client-side prediction
- **Preferences**: Default configuration and persistence for star protocol settings
- **XML parsing**: Support configuration via XML element parsing

## External Dependencies
- **NetworkGameProtocol.h**: Base class definition; provides abstract interface
- **config.h**: Build configuration (enables/disables networking)
- **stdio.h**: Standard I/O (preference file writing)
- **Forward declarations**: `XML_ElementParser`, `DDPPacketBuffer`, `NetTopology` (definitions elsewhere)
- **Network layer**: Implied lower-level DDP (Datagram Delivery Protocol) implementation

# Source_Files/Network/Update.cpp
## File Purpose
Implements online update checking for the Aleph One game engine. Spawns a background thread to connect to the SourceForge update server, fetch platform-specific version information via HTTP, and maintain the update availability status without blocking the main game loop.

## Core Responsibilities
- Manage singleton lifecycle for the Update checker
- Spawn and manage a background SDL thread for non-blocking network I/O
- Establish TCP connections to the update server (marathon.sourceforge.net)
- Construct and send HTTP GET requests for platform-specific update metadata
- Parse HTTP responses to extract version information (`A1_DATE_VERSION`, `A1_DISPLAY_VERSION`)
- Maintain and expose update status (checking, available, failed, none)

## External Dependencies
- **SDL_net:** `SDLNet_ResolveHost()`, `SDLNet_TCP_Open()`, `SDLNet_TCP_Send()`, `SDLNet_TCP_Recv()`
- **SDL_thread:** `SDL_CreateThread()`, `SDL_WaitThread()`
- **C standard library:** `strtok()`, `strncmp()`, `strlen()`, `sprintf()`
- **C++ standard library:** `std::string` (assign, compare, size, clear)
- **alephversion.h:** `A1_UPDATE_PLATFORM`, `A1_DATE_VERSION` (version constants)
- **boost/tokenizer.hpp:** included but not used in this file

**Security/Robustness Notes:**
- `sprintf()` without bounds checking (buffer overflow risk)
- No HTTPS/TLS support; HTTP only
- Hard-coded server address not configurable
- No timeout on socket operations (could hang indefinitely)
- No certificate validation or man-in-the-middle protection

# Source_Files/Network/Update.h
## File Purpose
Declares the `Update` singleton class for asynchronous online version checking in Aleph One. Manages update availability status and provides access to new version information when available.

## Core Responsibilities
- Implements singleton pattern for centralized update management
- Tracks update check status (checking, failed, available, unavailable)
- Runs version check asynchronously in a separate SDL thread
- Stores and exposes new version information
- Guards all functionality behind networking compile flag

## External Dependencies
- `#include <string>` ΓÇö `std::string` for version storage
- `#include <SDL_thread.h>` ΓÇö `SDL_Thread` for async execution
- `#include "cseries.h"` ΓÇö Aleph One utilities (likely assertion macros)
- `config.h` ΓÇö Conditional `DISABLE_NETWORKING` guard
- **Defined elsewhere**: Implementation of `StartUpdateCheck()`, `Thread()`, and actual HTTP networking (likely in `.cpp` counterpart)


