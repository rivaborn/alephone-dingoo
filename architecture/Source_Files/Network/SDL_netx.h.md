# Source_Files/Network/SDL_netx.h

## File Purpose
Header file providing UDP broadcast capabilities for SDL_net. Extends SDL_net with functions to enable broadcast sending across all available network interfaces, addressing functionality that base SDL_net lacks. Part of the Aleph One game engine networking subsystem.

## Core Responsibilities
- Enable/disable SO_BROADCAST socket option on UDP sockets
- Send UDP packets to broadcast addresses across all network interfaces
- Manage broadcast state and validate socket readiness

## Key Types / Data Structures
None (uses SDL_net types: `UDPsocket`, `UDPpacket`).

## Global / File-Static State
None.

## Key Functions / Methods

### SDLNetx_EnableBroadcast
- Signature: `int SDLNetx_EnableBroadcast(UDPsocket inSocket);`
- Purpose: Enable broadcast capability on a UDP socket so it can send to broadcast addresses.
- Inputs: `inSocket` ΓÇö UDP socket to enable
- Outputs/Return: Positive if broadcast enabled; 0 if failed
- Side effects: Modifies socket options (likely `SO_BROADCAST` flag)
- Calls: Not visible (implementation elsewhere)
- Notes: Must be called before `SDLNetx_UDP_Broadcast` will succeed

### SDLNetx_DisableBroadcast
- Signature: `int SDLNetx_DisableBroadcast(UDPsocket inSocket);`
- Purpose: Disable broadcast capability on a UDP socket.
- Inputs: `inSocket` ΓÇö UDP socket to disable
- Outputs/Return: Positive if broadcast disabled; 0 if failed
- Side effects: Modifies socket options
- Calls: Not visible (implementation elsewhere)
- Notes: Reverses `EnableBroadcast`

### SDLNetx_UDP_Broadcast
- Signature: `int SDLNetx_UDP_Broadcast(UDPsocket inSocket, UDPpacket* inPacket);`
- Purpose: Send a UDP packet to broadcast addresses on all available network interfaces capable of IP/UDP broadcast.
- Inputs: 
  - `inSocket` ΓÇö socket to send from (must have broadcast enabled via `EnableBroadcast`)
  - `inPacket` ΓÇö packet to broadcast (host-part of address ignored; port used)
- Outputs/Return: Number of interfaces packet was sent on (0 if not sent); `inPacket->status` contains result from last send
- Side effects: Sends network packets to multiple broadcast addresses; modifies `inPacket->status`
- Calls: Not visible (implementation elsewhere)
- Notes: Ignores socket "channels" configuration; iterates through all available interfaces

## Control Flow Notes
Not inferable from header. Expected to be used during:
- **Network init**: `EnableBroadcast()` on sockets destined for broadcast messages
- **Game loop / networking frame**: `SDLNetx_UDP_Broadcast()` to announce presence or distribute state
- **Shutdown**: `DisableBroadcast()` before socket closure

## External Dependencies
- `SDL_net.h` ΓÇö provides `UDPsocket`, `UDPpacket` types
- `config.h` ΓÇö guards code with `DISABLE_NETWORKING` preprocessor flag; requires `HAVE_SDL_NET`
