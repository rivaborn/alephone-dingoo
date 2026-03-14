# Source_Files/Network/SSLP_Protocol.h

## File Purpose
Defines the network protocol specification for SSLP (Simple Service Location Protocol), enabling service discovery on non-AppleTalk networks for Aleph One game servers. Specifies packet format, constants, and protocol semantics for finding and advertising network services via UDP broadcast/unicast.

## Core Responsibilities
- Define SSLP packet structure (`struct SSLP_Packet`) for network wire format
- Establish protocol constants: magic number, version, message types (FIND/HAVE/LOST)
- Configure network port and service type/name field constraints
- Document protocol behavior: broadcast discovery, unicast responses, loss tolerance, unsolicited announcements
- Ensure cross-platform binary compatibility through packed struct layout and big-endian field ordering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `struct SSLP_Packet` | struct | Network packet format carrying protocol magic, version, message type, service port, service type, and service name; laid out for binary wire transmission |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SSLP_PORT` | `#define` | global | UDP port for SSLP communication (default 15367) |
| `SSLPP_MAGIC` | `#define` | global | Protocol magic number (0x73736c70, 'sslp' in network byte order) |
| `SSLPP_VERSION` | `#define` | global | Protocol version (1); independent of Marathon game protocol version |
| `SSLPP_MESSAGE_FIND` | `#define` | global | Message type constant (0x66696e64, 'find') for service discovery queries |
| `SSLPP_MESSAGE_HAVE` | `#define` | global | Message type constant (0x68617665, 'have') for service advertisements |
| `SSLPP_MESSAGE_LOST` | `#define` | global | Message type constant (0x6c6f7374, 'lost') for service withdrawal |
| `SSLP_MAX_TYPE_LENGTH` | `#define` | global | Maximum service type string length (32 bytes) |
| `SSLP_MAX_NAME_LENGTH` | `#define` | global | Maximum service name string length (32 bytes) |
| `SIZEOF_SSLP_Packet` | `const int` | global | Packed wire size of SSLP_Packet (80 bytes); compiler-independent |

## Key Functions / Methods
None. This file is a protocol specification header; no executable functions defined.

## Control Flow Notes
Protocol operates as a three-phase service discovery mechanism over unreliable UDP:

1. **FIND Phase**: Client broadcasts FIND datagrams (rebroadcast every 1ΓÇô5 seconds) with desired `service_type`; may unicast to suspected hosts.
2. **HAVE Phase**: Server responds to each FIND with unicast HAVE (containing `service_name` and `sslpp_service_port`); may also broadcast unsolicited HAVE on service activation or unicast periodically if hinting targets.
3. **LOST Phase**: Server broadcasts or unicasts LOST when service becomes unavailable; not required but helps clients age stale entries.

No delivery guarantee; clients tolerate dropped packets through rebroadcast; servers tolerate missed FINDs through client retry loops.

## External Dependencies
- **SDL_net.h**: Cross-platform UDP/IP networking library (datagram support required)
- **config.h**: Build configuration; guards entire section with `DISABLE_NETWORKING`

## Notes
- Service type and name treated as C-strings in comparisons (null-terminated); stored fields may omit terminator if all 32 bytes used.
- Packet must be transmitted in network byte order (big-endian); packed struct layout ensures cross-platform binary compatibility.
- Designed for Aleph One player discovery during "Join Network Game" dialog; not intended as general-purpose service discovery protocol.
- Simplified relative to heavyweight service discovery protocols; no retry guarantees, authentication, or registration.
