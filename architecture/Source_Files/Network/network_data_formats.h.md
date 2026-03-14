# Source_Files/Network/network_data_formats.h

## File Purpose
Defines cross-platform wire-format structures and conversion functions for network packet serialization. Ensures consistent padding, byte ordering, and alignment across all platforms by providing paired structures (`_NET` for wire format and unpacked for platform format) with conversion utilities (`netcpy` overloads).

## Core Responsibilities
- Define `_NET` structures as binary-compatible wire formats for all network data types
- Provide bidirectional `netcpy()` conversion functions between platform and network formats
- Handle endianness conversion (byte-swapping) based on host architecture
- Define fixed-size constants (SIZEOF_*) for each wire structure
- Support cross-platform game networking without platform-specific code in consumers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| NetPacketHeader_NET | struct | Wire format for packet headers (6 bytes raw) |
| NetPacket_NET | struct | Wire format for action flag packets (variable size) |
| NetDistributionPacket_NET | struct | Wire format for generic distribution packets (6 bytes header) |
| IPaddress_NET | struct | Wire format for IP addresses (6 bytes) |
| network_audio_header_NET | struct | Wire format for network audio headers (8 bytes) |

## Global / File-Static State
None.

## Key Functions / Methods

### netcpy (NetPacketHeader_NET, bidirectional)
- Signature: `void netcpy(NetPacketHeader_NET* dest, const NetPacketHeader* src);` + reverse overload
- Purpose: Convert packet headers between wire and platform format
- Inputs: Source structure (either _NET or unpacked)
- Outputs: Destination structure in opposite format
- Side effects: Modifies destination buffer
- Calls: (Implementation in network_data_formats.cpp, not visible)

### netcpy (NetPacket_NET, bidirectional)
- Signature: `void netcpy(NetPacket_NET* dest, const NetPacket* src);` + reverse
- Purpose: Convert action flag packets
- Inputs: Source packet
- Outputs: Converted packet
- Side effects: Modifies destination
- Calls: (Implementation elsewhere)

### netcpy (uint32, bidirectional, length-parameterized)
- Signature: `void netcpy(uint32* dest, const uint32* src, size_t length);`
- Purpose: Convert 32-bit action flag arrays with endianness handling
- Inputs: Source uint32 array, length in bytes
- Outputs: Destination uint32 array
- Side effects: On little-endian: byte-swaps; on big-endian: memcpy
- Calls: `memcpy()` (on big-endian systems)
- Notes: Inline on big-endian (no-op); symmetric operation

### netcpy (IPaddress_NET, bidirectional)
- Signature: `void netcpy(IPaddress_NET* dest, const IPaddress* src);` + reverse
- Purpose: Convert IP address structures
- Inputs: Source structure
- Outputs: Destination structure
- Notes: Host and port fields NOT byte-swapped; always transmitted in network byte order

### netcpy (network_audio_header_NET, bidirectional)
- Signature: `void netcpy(network_audio_header_NET* dest, const network_audio_header* src);` + reverse
- Purpose: Convert network audio metadata headers
- Inputs: Source header
- Outputs: Destination header

## Control Flow Notes
This file provides primitives for the network protocol layer (initialization and frame updates). It's invoked whenever game state, action flags, or audio must be transmitted:
- **Startup**: converts game_info and player_info during gather/join
- **Frame**: converts action flags each ring packet (via uint32 netcpy)
- **Distribution**: converts game data, map chunks, and chat messages
- **Audio**: converts audio stream headers during voice chat

The file itself contains no control flow; consumers call `netcpy()` during packet construction/reception.

## External Dependencies
- `config.h` ΓÇö DISABLE_NETWORKING guard; version/feature flags
- `cseries.h` ΓÇö endianness constant ALEPHONE_LITTLE_ENDIAN; integer types (uint8, uint32, int16, int32)
- `network.h` ΓÇö game_info, player_info type definitions (used by unpacked structures defined elsewhere)
- `network_private.h` ΓÇö NetPacketHeader, NetPacket, NetDistributionPacket (unpacked struct definitions)
- `network_audio_shared.h` ΓÇö network_audio_header (unpacked struct)
- C standard library ΓÇö memcpy() (called inline on big-endian)
