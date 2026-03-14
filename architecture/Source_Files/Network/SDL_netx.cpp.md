# Source_Files/Network/SDL_netx.cpp

## File Purpose
Extends SDL_net with UDP broadcast capabilities not natively supported by the library. Provides platform-abstracted functions to enable broadcast mode on sockets and send packets to all available network broadcast addresses. Handles platform-specific variations (Windows, BeOS, Mac, Linux/BSD).

## Core Responsibilities
- Enable/disable the SO_BROADCAST socket option on UDP sockets
- Enumerate network interfaces and their broadcast addresses (Unix/BSD platforms)
- Send UDP packets to all enumerated broadcast addresses or a platform-appropriate fallback
- Abstract platform differences (Windows accepts 255.255.255.255; Unix requires explicit interface enumeration)
- Manage cached broadcast address list to avoid repeated system calls

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `UDPsocket` | typedef (from SDL_net) | Opaque handle to UDP socket; code extracts raw socket FD via pointer arithmetic |
| `UDPpacket` | struct (from SDL_net) | UDP packet with address (host/port) and payload |
| `struct ifreq` | system struct | Interface request; holds interface name and address |
| `struct ifconf` | system struct | Interface configuration query; buffer of `ifreq` entries |
| `struct sockaddr_in` | system struct | IPv4 socket address (used to extract broadcast address) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sCollectedBroadcastAddresses` | bool | static | Flag: set to `true` after first enumeration attempt |
| `sNumberOfBroadcastAddresses` | int | static | Count of valid broadcast addresses found |
| `sBroadcastAddresses` | long[8] | static | Array of broadcast host addresses (port added at send time) |
| `kMaxNumBroadcastAddresses` | int (const) | static | Max interfaces to enumerate (8) |
| `kIFConfigBufferSize` | int (const) | static | Buffer for interface enumeration (1024 bytes) |

## Key Functions / Methods

### SDLNetx_EnableBroadcast
- **Signature:** `int SDLNetx_EnableBroadcast(UDPsocket inSocket)`
- **Purpose:** Enable broadcast option on the socket so packets can be sent to broadcast addresses.
- **Inputs:** `inSocket` ΓÇö UDP socket to enable
- **Outputs/Return:** Returns non-zero (value of SO_BROADCAST option) on success; 0 on failure or unsupported platform
- **Side effects:** Calls `setsockopt()` on underlying socket FD; on non-Windows platforms, triggers `SDLNetxint_CollectBroadcastAddresses()` once
- **Calls:** `SDLNetxint_CollectBroadcastAddresses()` (lazy initialization)
- **Notes:** Extracts raw socket FD via pointer arithmetic on `UDPsocket` (assumes offset 1 in SDL_net struct); returns early on BeOS/Metrowerks

### SDLNetx_DisableBroadcast
- **Signature:** `int SDLNetx_DisableBroadcast(UDPsocket inSocket)`
- **Purpose:** Disable broadcast option on the socket.
- **Inputs:** `inSocket` ΓÇö UDP socket to disable
- **Outputs/Return:** Returns 1 on success; 0 on failure or unsupported platform
- **Side effects:** Calls `setsockopt(SO_BROADCAST, 0)` on underlying socket
- **Calls:** (none)
- **Notes:** Mirror of `EnableBroadcast()` with opposite logic

### SDLNetx_UDP_Broadcast (Unix/BSD implementation)
- **Signature:** `int SDLNetx_UDP_Broadcast(UDPsocket inSocket, UDPpacket* inPacket)`
- **Purpose:** Send packet to all enumerated broadcast addresses.
- **Inputs:** `inSocket` ΓÇö socket; `inPacket` ΓÇö packet to send (port preserved, host overwritten)
- **Outputs/Return:** Count of interfaces packet was successfully sent on
- **Side effects:** Temporarily modifies `inPacket->address.host`; restores after sending
- **Calls:** `SDLNet_UDP_Send()` for each broadcast address
- **Notes:** Saves/restores original host to avoid trampling caller's address

### SDLNetx_UDP_Broadcast (Windows/BeOS/Mac implementation)
- **Signature:** `int SDLNetx_UDP_Broadcast(UDPsocket inSocket, UDPpacket* inPacket)`
- **Purpose:** Send packet to single broadcast address (platform accepts 255.255.255.255).
- **Inputs:** `inSocket` ΓÇö socket; `inPacket` ΓÇö packet to send
- **Outputs/Return:** Result of single `SDLNet_UDP_Send()` call (0 or 1)
- **Side effects:** Temporarily sets host to 0xffffffff; restores after send
- **Calls:** `SDLNet_UDP_Send()`
- **Notes:** Simpler fallback for platforms that accept all-ones address

### SDLNetxint_CollectBroadcastAddresses (internal, Unix/BSD only)
- **Signature:** `int SDLNetxint_CollectBroadcastAddresses(UDPsocket inSocket)`
- **Purpose:** Enumerate all IP-capable network interfaces and cache their broadcast addresses.
- **Inputs:** `inSocket` ΓÇö socket used to issue ioctl requests
- **Outputs/Return:** Number of broadcast addresses collected
- **Side effects:** Populates static arrays `sBroadcastAddresses[]` and `sNumberOfBroadcastAddresses`; sets `sCollectedBroadcastAddresses = true`
- **Calls:** `ioctl(SIOCGIFCONF)`, `ioctl(SIOCGIFBRDADDR)`
- **Notes:** Uses `_SIZEOF_ADDR_IFREQ()` macro (Mac OS X) to account for variable-length address structures; falls back to `sizeof(ifreq)` if not available. Stops collecting if buffer fills or max address count reached.

## Control Flow Notes
This is a utility library used on-demand during network initialization or when broadcast packets are needed. Not part of main game loop. On Unix/BSD, the first call to `EnableBroadcast()` triggers interface enumeration (cached for lifetime). Subsequent calls are cheap. Windows/BeOS/Mac skip enumeration and use simplified path.

## External Dependencies
- **SDL_net:** `UDPsocket`, `UDPpacket`, `SDLNet_UDP_Send()`
- **System socket APIs:** `setsockopt()`, `ioctl()`, `socket.h`, `netinet/in.h`, `net/if.h`
- **Platform-specific headers:** `winsock.h` (Windows), system headers vary by OS
- **Compiler/platform conditionals:** `WIN32`, `__BEOS__`, `mac`, `__MWERKS__`, `__svr4__`
