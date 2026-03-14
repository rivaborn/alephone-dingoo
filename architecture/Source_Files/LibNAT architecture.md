# Subsystem Overview

## Purpose
LibNAT is a UPnP library that discovers and controls Internet Gateway Devices (IGDs) on the local network. It enables NAT traversal by querying public IP addresses and managing port mappings (TCP/UDP), allowing the game to establish network connectivity behind NAT firewalls for peer-to-peer features.

## Key Files
| File | Role |
|------|------|
| `libnat.h` | Public API: IGD discovery, port mapping management, public IP queries, controller lifecycle |
| `error.h` | Error/status code constants and `LNat_Print_Internal_Error()` utility for diagnostic reporting |
| `http.h` | HTTP GET/POST request generation, header management, execution, and response/buffer memory handling |
| `ssdp.h` | SSDP discovery protocol interface for locating UPnP devices on the network |
| `os.h` | Platform detection and function aliasing macros; maps generic `LNat_Os_*` names to Unix or Windows implementations |
| `os_common.h` | Common socket operation declarations and conditional platform-specific header inclusion (fcntl/winsock) |
| `utility.h` | Macro constants (HTTP status codes, protocol strings, network size limits), ASCII string case conversion |

## Core Responsibilities
- Discover UPnP-enabled IGDs on the local network via SSDP
- Establish and remove TCP/UDP port mappings on discovered IGDs
- Query public IP address from IGDs
- Manage IGD controller object lifecycle
- Provide platform-agnostic UDP and TCP socket operations (Unix and Windows abstraction)
- Execute HTTP GET and POST requests for UPnP control communication
- Propagate errors through status codes and diagnostic output
- Define and enforce network buffer sizes, HTTP protocol constants, and string handling utilities

## Key Interfaces & Data Flow
**Exposes:**
- `libnat.h` public API (discovery, port mapping CRUD, IP queries, error reporting)

**Consumes:**
- OS socket layer implementations (Unix: fcntl, socket, select; Windows: winsock)
- HTTP layer (for SOAP/UPnP control messages to routers)
- SSDP layer (for device discovery over UDP)
- XML parser (referenced in libnat.h for UPnP device description parsing)

## Runtime Role
Inferred to be called during network initialization to discover and configure NAT traversal before game networking is available; no frame or shutdown lifecycle details provided in documentation.

## Notable Implementation Details
- **Platform abstraction:** `OsSocket` opaque wrapper hides OS-specific socket types (int for Unix, SOCKET for Windows)
- **Cross-platform macro aliasing:** `os.h` redirects `LNat_Os_*` function calls to platform-specific implementations
- **Memory contracts:** Callers responsible for freeing SSDP response buffers; HTTP functions manage request/response lifecycle internally
- **Error categories:** Distinct error codes for socket, HTTP, SSDP, and UPnP failures to enable targeted diagnostics
