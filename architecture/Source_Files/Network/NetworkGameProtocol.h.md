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

## Key Types / Data Structures
None (pure abstract interface class).

## Global / File-Static State
None.

## Key Functions / Methods

### Enter
- Signature: `virtual bool Enter(short* inNetStatePtr) = 0`
- Purpose: Initialize and join the network session
- Inputs: Pointer to network state variable
- Outputs/Return: Boolean success/failure
- Side effects: Initializes network protocol state
- Calls: (abstract; implementation-specific)
- Notes: Must be called before gameplay begins

### Exit1 / Exit2
- Signature: `virtual void Exit1() = 0; virtual void Exit2() = 0`
- Purpose: Two-phase shutdown protocol
- Inputs: None
- Outputs/Return: Void
- Side effects: Cleans up network resources
- Notes: Two phases allow graceful disconnection before resource release

### Sync
- Signature: `virtual bool Sync(NetTopology* inTopology, int32 inSmallestGameTick, size_t inLocalPlayerIndex, size_t inServerPlayerIndex) = 0`
- Purpose: Synchronize game state across all network participants
- Inputs: Network topology, minimum game tick, local and server player indices
- Outputs/Return: Boolean success/failure
- Calls: (abstract; implementation-specific)
- Notes: Called before gameplay begins; critical for ensuring all players start in consistent state

### UnSync
- Signature: `virtual bool UnSync(bool inGraceful, int32 inSmallestPostgameTick) = 0`
- Purpose: End game synchronization gracefully or forcefully
- Inputs: Graceful flag, smallest postgame tick
- Outputs/Return: Boolean success/failure
- Notes: Invoked at game end; graceful mode allows proper cleanup

### DistributeInformation
- Signature: `virtual void DistributeInformation(short type, void *buffer, short buffer_size, bool send_to_self, bool only_send_to_team) = 0`
- Purpose: Broadcast game data to network participants
- Inputs: Data type identifier, buffer pointer, buffer size, self-send flag, team-only flag
- Outputs/Return: Void
- Side effects: Sends network packets to other players
- Notes: Flexible distribution targeting (team-only or all players)

### PacketHandler
- Signature: `virtual void PacketHandler(DDPPacketBuffer* inPacket) = 0`
- Purpose: Process incoming network packets
- Inputs: Incoming packet buffer
- Outputs/Return: Void
- Side effects: Updates local game state based on packet contents
- Notes: Called by network dispatcher for each received packet

### GetNetTime
- Signature: `virtual int32 GetNetTime() = 0`
- Purpose: Retrieve current synchronized network time
- Outputs/Return: Current network tick
- Notes: Used for frame-independent game logic across network

### Unconfirmed Action Flags (3 methods)
- `GetUnconfirmedActionFlagsCount()`: Return pending action flags not yet authoritative
- `PeekUnconfirmedActionFlag(int32 offset)`: Read action flag at given offset
- `UpdateUnconfirmedActionFlags()`: Commit pending action flags to official state
- Purpose: Support client-side prediction while awaiting server confirmation
- Notes: Enables smooth gameplay despite network latency

## Control Flow Notes
Typical protocol lifecycle: **Enter ΓåÆ (Sync ΓåÆ repeat [DistributeInformation/PacketHandler during game loop] ΓåÆ UnSync) ΓåÆ Exit1/Exit2**.
Action flags follow a lookahead/prediction model: unconfirmed flags are generated locally, then updated as authoritative data arrives from the server.

## External Dependencies
- `network_private.h` ΓÇö Provides `NetTopology`, `DDPPacketBuffer` structures and network constants
- `config.h` ΓÇö Compile-time configuration (DISABLE_NETWORKING guard)
