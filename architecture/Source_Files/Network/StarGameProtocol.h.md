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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| StarGameProtocol | class | Concrete implementation of star-topology network protocol; inherits from NetworkGameProtocol |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| (extern functions) | DefaultStarPreferences, WriteStarPreferences | global | Configuration initialization and persistence for star protocol parameters |

## Key Functions / Methods

### Enter
- Signature: `bool Enter(short* inNetStatePtr)`
- Purpose: Initialize the star protocol and join the game network
- Inputs: Pointer to network state variable
- Outputs/Return: Boolean success/failure
- Side effects: Initializes protocol state, likely allocates resources
- Calls: (Implementation in .cpp file, not visible here)
- Notes: Part of lifecycle; must be called before game play begins

### Exit1 / Exit2
- Signature: `void Exit1()`, `void Exit2()`
- Purpose: Two-phase exit mechanism for graceful shutdown
- Inputs: None
- Outputs/Return: None
- Side effects: Tears down protocol state; Exit1 likely initiates, Exit2 finalizes
- Calls: (Implementation in .cpp file)
- Notes: Two-phase design suggests cleanup of both local and network resources

### DistributeInformation
- Signature: `void DistributeInformation(short type, void *buffer, short buffer_size, bool send_to_self, bool only_send_to_team)`
- Purpose: Broadcast a message to players on the network
- Inputs: Message type, payload buffer, buffer size, self-delivery flag, team-only flag
- Outputs/Return: None (fire-and-forget)
- Side effects: Sends packet(s) over network
- Calls: (Network layer, not visible)
- Notes: Supports selective routing (self, team, broadcast)

### Sync
- Signature: `bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex)`
- Purpose: Synchronize game state across the network
- Inputs: Network topology, game tick floor, local and server player indices
- Outputs/Return: Boolean success/failure
- Side effects: Adjusts local game state to match network consensus
- Calls: (Implementation in .cpp)
- Notes: Critical for multi-player consistency; called periodically during gameplay

### UnSync
- Signature: `bool UnSync(bool inGraceful, int32 inSmallestPostgameTick)`
- Purpose: Desynchronize and clean up after game ends
- Inputs: Graceful flag, post-game tick threshold
- Outputs/Return: Boolean success/failure
- Side effects: Releases synchronization resources
- Calls: (Implementation in .cpp)

### GetNetTime
- Signature: `int32 GetNetTime()`
- Purpose: Retrieve the current network-synchronized time
- Inputs: None
- Outputs/Return: Network time as 32-bit integer
- Side effects: None
- Calls: (Network clock, not visible)

### PacketHandler
- Signature: `void PacketHandler(DDPPacketBuffer* inPacket)`
- Purpose: Handle an incoming network packet
- Inputs: Pointer to DDP packet buffer
- Outputs/Return: None
- Side effects: Updates protocol state based on packet contents; may trigger cascading updates
- Calls: (Packet parsing, not visible)
- Notes: Called from network layer when packets arrive

### GetUnconfirmedActionFlagsCount / PeekUnconfirmedActionFlag / UpdateUnconfirmedActionFlags
- Signature: `int32 GetUnconfirmedActionFlagsCount()`, `uint32 PeekUnconfirmedActionFlag(int32 offset)`, `void UpdateUnconfirmedActionFlags()`
- Purpose: Support client-side action prediction before server confirmation
- Inputs: offset for peeking
- Outputs/Return: Count and flag values
- Side effects: UpdateUnconfirmedActionFlags refreshes prediction cache
- Calls: (Internal prediction state)
- Notes: Allows smooth client-side gameplay while awaiting server authority

### GetParser
- Signature: `static XML_ElementParser* GetParser()`
- Purpose: Provide an XML parser for protocol configuration
- Inputs: None
- Outputs/Return: Pointer to XML element parser
- Side effects: None (static)
- Calls: (XML parsing, not visible)

## Control Flow Notes
**Lifecycle**: Enter ΓåÆ (Frame loop with Sync, PacketHandler, DistributeInformation) ΓåÆ UnSync ΓåÆ Exit1 ΓåÆ Exit2. The protocol is event-driven during the frame loop, responding to incoming packets and distributing local information periodically. Action flags prediction happens inline during the frame loop to minimize client-side latency.

## External Dependencies
- **NetworkGameProtocol.h**: Base class definition; provides abstract interface
- **config.h**: Build configuration (enables/disables networking)
- **stdio.h**: Standard I/O (preference file writing)
- **Forward declarations**: `XML_ElementParser`, `DDPPacketBuffer`, `NetTopology` (definitions elsewhere)
- **Network layer**: Implied lower-level DDP (Datagram Delivery Protocol) implementation
