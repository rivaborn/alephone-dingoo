# Source_Files/LibNAT/error.h
## File Purpose
Defines error and status codes returned by LibNAT networking library functions. Provides a centralized set of error constants (memory allocation, socket, HTTP, SSDP, UPnP failures) and an error printing utility for callers to diagnose failures.

## Core Responsibilities
- Define error/status code constants for all LibNAT subsystems
- Enable error propagation through function call stacks
- Provide human-readable error reporting via `LNat_Print_Internal_Error()`
- Distinguish error categories (generic, socket, HTTP, SSDP, UPnP)

## External Dependencies
None. Self-contained header with no external includes; implementation of `LNat_Print_Internal_Error()` is defined elsewhere in LibNAT.

# Source_Files/LibNAT/http.h
## File Purpose
Header file declaring HTTP client functions for sending GET and POST requests to a router. Provides a high-level interface for constructing HTTP messages, adding headers, executing requests, and managing memory for request/response structures.

## Core Responsibilities
- Generate and destroy HTTP GET request structures
- Generate and destroy HTTP POST request structures with bodies
- Add Request Header Fields and Entity Header Fields to POST messages
- Execute HTTP GET and POST requests to remote servers
- Manage memory allocation/deallocation for request and response buffers

## External Dependencies
- Standard C library (likely `stdio.h`, memory functions)
- Network/socket library (implementation file not provided)
- Likely uses UPnP/NAT library context ("LibNAT" suggests NAT traversal utilities)

# Source_Files/LibNAT/libnat.h
## File Purpose
Public API header for LibNAT, a UPnP library for discovering and controlling Internet Gateway Devices. Provides functions for NAT traversal, port mapping, and retrieving public IP addresses to enable network connectivity behind NAT firewalls (e.g., for peer-to-peer file transfers).

## Core Responsibilities
- Discover UPnP-enabled IGDs on the local network
- Query public IP address from discovered IGDs
- Establish and remove port mappings on IGDs for TCP/UDP protocols
- Manage lifecycle of controller objects
- Report library errors to the caller

## External Dependencies
- **Includes:** `"error.h"` ΓÇö defines all error code constants and `LNat_Print_Internal_Error()`.
- **Conditionally compiled:** `LIBNAT_API` macro for DLL export (currently disabled via `#if 0`).
- **C++ compatibility:** Wrapped in `extern "C"` block.
- **UPnP protocol:** Implementation (not in this file) must perform SSDP discovery, XML parsing, and SOAP communication; likely depends on socket, HTTP, and XML parsing libraries.

# Source_Files/LibNAT/os.h
## File Purpose
Provides platform-agnostic abstraction for network socket operations across Unix and Windows systems. Uses compile-time feature detection and macro redirection to present a unified API regardless of the underlying OS.

## Core Responsibilities
- Detect compilation platform (Unix vs Windows) via preprocessor conditionals
- Provide macro aliases mapping generic `LNat_Os_*` names to platform-specific implementations
- Declare opaque socket type (`OsSocket`) to hide OS-specific details
- Define function prototypes for UDP socket operations (setup, close, send, recv)
- Define function prototypes for TCP socket operations (connect, close, send, recv)
- Declare utility function to retrieve local IP address from an active socket

## External Dependencies
- No includes in this file (purely declarations)
- Implementation files (`os.c` and platform-specific variants) provide actual socket and network code
- Depends on platform headers (`winsock2.h` for Windows, `sys/socket.h` for Unix) in implementation

# Source_Files/LibNAT/os_common.h
## File Purpose
Common header file that abstracts operating system-specific socket operations across POSIX and Windows platforms. Declares the platform-agnostic socket interface functions and conditionally defines OS-specific socket structures. Serves as the contract that platform-specific implementations (Unix/Windows) must fulfill.

## Core Responsibilities
- Conditionally include platform-specific headers (Unix: fcntl, socket, select; Windows: winsock)
- Define the `OsSocket` opaque wrapper struct (int for Unix, SOCKET for Windows)
- Declare UDP socket operations: setup, send, recv, close
- Declare TCP socket operations: connect, send, recv, close
- Declare socket readiness polling functions (select-based blocking)
- Declare socket utility functions (address initialization, local IP lookup)
- Forward declare system structures (`sockaddr_in`, `hostent`) needed by setup functions

## External Dependencies
- `"os.h"` ΓÇô Platform detection and function aliasing macros
- **Unix**: `<fcntl.h>`, `<sys/socket.h>`, `<sys/select.h>`, `<netinet/in.h>`, `<arpa/inet.h>`, `<netdb.h>`, `<unistd.h>`, `<sys/time.h>`
- **Windows**: `<windows.h>`, `<winsock.h>`
- Standard C: `<stdlib.h>`, `<string.h>`, `<time.h>`
- Forward declarations: `struct sockaddr_in`, `struct hostent` (defined in system headers)

# Source_Files/LibNAT/ssdp.h
## File Purpose
Header file declaring the SSDP (Simple Service Discovery Protocol) interface for LibNAT. Provides a function to send SSDP discover requests to network devices (typically routers) and receive their responses for device discovery and NAT traversal purposes.

## Core Responsibilities
- Declare the SSDP discovery entry point
- Define the interface for sending SSDP requests over the network
- Specify memory management contract for responses (caller frees)

## External Dependencies
- Standard C library (`free()` mentioned in comments)
- SSDP/UPnP network protocol (implementation elsewhere)

# Source_Files/LibNAT/utility.h
## File Purpose
Header file declaring utility functions and compile-time constants used throughout the LibNAT project. Provides macro definitions for string processing, HTTP protocol constants, and network-related size limits, plus a function declaration for string case conversion.

## Core Responsibilities
- Define macro constants for string null-termination and buffer sizing
- Define HTTP protocol constants (status codes, protocol string, default port)
- Define maximum size constraints for network strings (URLs, hostnames, resources, ports)
- Declare the `LNat_Str_To_Upper` function for ASCII uppercase conversion
- Establish common constants used across the project to avoid magic numbers

## External Dependencies
- Standard C library (implementation uses standard string/character functions, not visible here)


