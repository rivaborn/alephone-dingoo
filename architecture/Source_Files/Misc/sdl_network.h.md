# Source_Files/Misc/sdl_network.h

## File Purpose
SDL implementation header for Aleph One's networking layer, providing abstractions over SDL_net for both UDP (DDP) and TCP (ADSP) operations. Defines data structures and function prototypes for packet handling, socket management, and network callbacks.

## Core Responsibilities
- Define network packet and frame structures for DDP (UDP-based datagram protocol)
- Provide TCP connection structures wrapping SDL sockets
- Declare type definitions for network addresses, entity names, and callback function pointers
- Declare DDP socket lifecycle and packet transmission functions
- Define constants for maximum packet/datagram sizes
- Provide filter and update callback interfaces for network entity lookups

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `NetEntityName` | typedef | 32-byte entity name storage (array of char) |
| `NetAddrBlock` | typedef | Network address wrapper (SDL's IPaddress type) |
| `DDPFrame` | struct | Outgoing UDP packet: size, data buffer, socket handle |
| `DDPPacketBuffer` | struct | Incoming UDP packet: protocol type, source address, datagram data |
| `ConnectionEnd` | struct | TCP connection: client socket, server socket, and socket set |
| `lookupUpdateProcPtr` | typedef | Callback: `(message, index) ΓåÆ void` for entity lookup updates |
| `lookupFilterProcPtr` | typedef | Callback: `(entity, address) ΓåÆ bool` for entity filtering |
| `PacketHandlerProcPtr` | typedef | Callback: `(packet) ΓåÆ void` for incoming DDP packet handling |

## Global / File-Static State
None.

## Key Functions / Methods

### NetState
- Signature: `short NetState(void);`
- Purpose: Query current network state
- Inputs: None
- Outputs/Return: Network state code (short)
- Side effects: None inferable
- Calls: None visible in this file
- Notes: Implementation in NETWORK.C; state codes defined in network.h

### NetSetServerIdentifier
- Signature: `void NetSetServerIdentifier(short identifier);`
- Purpose: Configure server identifier for network initialization
- Inputs: `identifier` ΓÇô server ID (short)
- Outputs/Return: None
- Side effects: Sets global/static network state
- Calls: None visible
- Notes: Used during network setup

### NetDDPOpenSocket
- Signature: `OSErr NetDDPOpenSocket(short *ioPortNumber, PacketHandlerProcPtr packetHandler);`
- Purpose: Open a UDP socket and register packet handler
- Inputs: `ioPortNumber` ΓÇô port (network byte order, in/out); `packetHandler` ΓÇô callback for incoming packets
- Outputs/Return: Error code; `ioPortNumber` updated with assigned port
- Side effects: Allocates UDP socket, registers callback
- Calls: None visible
- Notes: Only one UDP socket supported currently; original API passed socket numbers like Unix file descriptors, now repurposed for IP port numbers

### NetDDPSendFrame
- Signature: `OSErr NetDDPSendFrame(DDPFramePtr frame, NetAddrBlock *address, short protocolType, short socket);`
- Purpose: Transmit a UDP datagram
- Inputs: `frame` ΓÇô packet to send; `address` ΓÇô destination; `protocolType` ΓÇô protocol ID; `socket` ΓÇô port (currently unused, API legacy)
- Outputs/Return: Error code
- Side effects: Network I/O
- Calls: None visible
- Notes: Callers should use `NetDDPNewFrame()` to allocate frames

### NetDDPNewFrame / NetDDPDisposeFrame
- Signatures: `DDPFramePtr NetDDPNewFrame(void);` and `void NetDDPDisposeFrame(DDPFramePtr frame);`
- Purpose: Allocate and deallocate UDP frame structures
- Inputs/Outputs: Frame pointer
- Side effects: Memory allocation/deallocation
- Notes: Trivial helpers; frame size fixed at `ddpMaxData` (1500 bytes)

## Control Flow Notes
This file is the SDL-specific network backend. It initializes conditionally if `DISABLE_NETWORKING` is not set. DDP functions (UDP) handle unreliable datagram delivery; ConnectionEnd structures support TCP for reliable connections (ADSP emulation). The file provides low-level socket and packet abstractions; higher-level game logic uses interfaces in `network.h`.

## External Dependencies
- **SDL_net.h** ΓÇô SDL networking library (IPaddress, UDPsocket, TCPsocket, SDLNet_SocketSet types)
- **network.h** ΓÇô Higher-level network interface and game state enums
- **cseries.h** ΓÇô Common utilities (uint16, byte, etc.)
- **config.h** ΓÇô Build configuration and feature flags
