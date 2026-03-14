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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| NetPacketHeader | struct | Wraps all datagram packets with tag and sequence number |
| NetPacket | struct | Ring protocol packet carrying action flags from players |
| NetDistributionPacket | struct | Generic packet for non-action-flag data distribution |
| NetPlayer | struct | Player info within a topology: address, identifier, stream ID, net_dead flag, player_data |
| NetTopology | struct | Complete network topology: tag, player list, next identifier, game_data |
| NetChatMessage | struct | Chat text message with sender identifier and text buffer (250 chars) |
| NetDistributionInfo | struct | Metadata for a distribution type: lossy flag and callback proc |
| GathererAvailableAnnouncer | class | Manages SSLP service instance to advertise gatherer availability |
| JoinerSeekingGathererAnnouncer | class | Manages SSLP service discovery to locate available gatherers |
| ClientChatInfo | struct | Chat metadata: player name, color, team |

## Global / File-Static State
None.

## Key Functions / Methods

### NetGetDistributionInfoForType
- Signature: `const NetDistributionInfo* NetGetDistributionInfoForType(int16 inType);`
- Purpose: Retrieve the distribution handler (lossy flag and callback) for a given distribution type ID.
- Inputs: `inType` ΓÇô distribution type identifier
- Outputs/Return: Pointer to NetDistributionInfo or NULL if type unknown
- Side effects: None inferable
- Calls: Not defined in this file (defined elsewhere in network_data_formats.cpp or similar)
- Notes: Used to route incoming distribution packets to the correct handler callback

### GathererAvailableAnnouncer::GathererAvailableAnnouncer
- Signature: `GathererAvailableAnnouncer();`
- Purpose: Constructor; initialize and start advertising gatherer service via SSLP
- Inputs: None
- Outputs/Return: None
- Side effects: Creates SSLP_ServiceInstance, calls SSLP_Allow_Service_Discovery
- Calls: SSLP_Allow_Service_Discovery (from SSLP_API.h)
- Notes: Manages a single static service instance for advertisement

### GathererAvailableAnnouncer::~GathererAvailableAnnouncer
- Signature: `~GathererAvailableAnnouncer();`
- Purpose: Destructor; stop advertising gatherer service
- Inputs: None
- Outputs/Return: None
- Side effects: Calls SSLP_Disallow_Service_Discovery
- Calls: SSLP_Disallow_Service_Discovery
- Notes: Cleans up SSLP_ServiceInstance

### GathererAvailableAnnouncer::pump
- Signature: `static void pump();`
- Purpose: Periodic pump to refresh SSLP announcements; called by game loop
- Inputs: None
- Outputs/Return: None
- Side effects: Updates SSLP broadcast/hinting state
- Calls: SSLP_Pump (from SSLP_API.h)
- Notes: Must be called frequently (~ every frame or at least once per second)

### JoinerSeekingGathererAnnouncer::JoinerSeekingGathererAnnouncer
- Signature: `JoinerSeekingGathererAnnouncer(bool shouldSeek);`
- Purpose: Constructor; optionally start searching for available gatherers via SSLP
- Inputs: `shouldSeek` ΓÇô if true, begin SSLP service discovery for gatherers
- Outputs/Return: None
- Side effects: If shouldSeek true, calls SSLP_Locate_Service_Instances with found/lost callbacks
- Calls: SSLP_Locate_Service_Instances
- Notes: Callbacks are static methods; only one joiner-search active at a time (per SSLP API limits)

### JoinerSeekingGathererAnnouncer::~JoinerSeekingGathererAnnouncer
- Signature: `~JoinerSeekingGathererAnnouncer();`
- Purpose: Destructor; stop searching for gatherers
- Inputs: None
- Outputs/Return: None
- Side effects: Calls SSLP_Stop_Locating_Service_Instances
- Calls: SSLP_Stop_Locating_Service_Instances
- Notes: Invalidates any pending service instances

### JoinerSeekingGathererAnnouncer::pump
- Signature: `static void pump();`
- Purpose: Periodic pump for SSLP discovery updates; called by game loop
- Inputs: None
- Outputs/Return: None
- Side effects: Processes SSLP callbacks (found_gatherer_callback, lost_gatherer_callback)
- Calls: SSLP_Pump
- Notes: Must be called frequently for responsive discovery

### JoinerSeekingGathererAnnouncer::found_gatherer_callback
- Signature: `static void found_gatherer_callback(const SSLP_ServiceInstance* instance);`
- Purpose: SSLP callback invoked when a gatherer service instance is discovered
- Inputs: `instance` ΓÇô discovered SSLP service instance (valid until lost_gatherer_callback)
- Outputs/Return: None
- Side effects: Likely updates internal gatherer list; may trigger UI update or auto-join logic
- Calls: Not visible in this file
- Notes: May be invoked from a different thread; must be thread-safe

### JoinerSeekingGathererAnnouncer::lost_gatherer_callback
- Signature: `static void lost_gatherer_callback(const SSLP_ServiceInstance* instance);`
- Purpose: SSLP callback invoked when a previously discovered gatherer becomes unavailable
- Inputs: `instance` ΓÇô pointer to the lost service instance (invalid after callback returns)
- Outputs/Return: None
- Side effects: Likely removes gatherer from list; may update UI or cancel pending join
- Calls: Not visible in this file
- Notes: After this callback, the instance pointer is no longer valid

## Control Flow Notes
This file defines the data structures and types used throughout the network subsystem. It is not itself part of the control flow but rather provides the foundational types for:
- **Init phase**: GathererAvailableAnnouncer/JoinerSeekingGathererAnnouncer constructors are called to start service discovery
- **Frame/Update phase**: `pump()` methods are called frequently to process SSLP updates and drive discovery
- **Shutdown phase**: Destructors are called to clean up SSLP resources
- **In-game**: NetPacket and NetDistributionPacket structures are used in the ring protocol for per-frame synchronization

## External Dependencies
- **config.h** ΓÇô build-time configuration flags (e.g., DISABLE_NETWORKING)
- **cstypes.h** ΓÇô platform-specific integer types (int16, int32, uint8, uint32, etc.)
- **sdl_network.h** ΓÇô SDL networking definitions (IPaddress, TCPsocket, UDPsocket, etc.)
- **network.h** ΓÇô public network API and callback interfaces (game_info, player_info, NetDistributionProc, etc.)
- **SSLP_API.h** ΓÇô service discovery API (SSLP_ServiceInstance, SSLP_Allow_Service_Discovery, etc.)
- **memory** (C++ standard library) ΓÇô for std::string in ClientChatInfo
- **Defined elsewhere**: NetGetDistributionInfoForType, get_network_version(), various ring-protocol implementations
