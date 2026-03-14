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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OsSocket | struct | Platform-agnostic opaque wrapper around OS socket handle (int on Unix, SOCKET on Windows) |

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Common_Socket_Udp_Send
- Signature: `int LNat_Common_Socket_Udp_Send(OsSocket * s, const char * host_addr, short int port, char * buf, int amt, int * amt_sent)`
- Purpose: Send data over a UDP socket to a specified address and port
- Inputs: socket, target host address (string), port, data buffer, amount to send
- Outputs/Return: status code; writes actual bytes sent to `amt_sent` pointer
- Side effects: Network I/O

### LNat_Common_Socket_Udp_Recv
- Signature: `int LNat_Common_Socket_Udp_Recv(OsSocket * s, const char * host_addr, short int port, char * buf, int amt, int * amt_recv, int timeout_sec)`
- Purpose: Receive data over a UDP socket with optional timeout
- Inputs: socket, expected source address, port, buffer, expected size, timeout in seconds
- Outputs/Return: status code; writes actual bytes received to `amt_recv` pointer
- Side effects: Network I/O, blocking up to timeout

### LNat_Common_Socket_Send / LNat_Common_Socket_Recv
- Purpose: Send/receive data over a connected (TCP) socket
- Inputs: socket, buffer, amount, optional timeout for recv
- Outputs/Return: status code; actual bytes transferred via pointer parameter
- Side effects: Network I/O, blocking on recv

### Select_Till_Readyread / Select_Till_Readywrite
- Purpose: Block until socket becomes readable or writable, or timeout expires
- Inputs: socket, timeout in seconds
- Outputs/Return: status code
- Side effects: Blocking syscall (select)

### Common_Initialize_Sockaddr_in
- Purpose: Initialize a sockaddr_in structure for network operations
- Inputs: sockaddr_in pointer, hostent pointer (output), host address string, port
- Outputs/Return: status code; populated hostent and sockaddr_in structures
- Side effects: DNS lookup (gethostbyname)

### LNat_Common_Get_Local_Ip
- Purpose: Retrieve the local IP address from a connected socket
- Inputs: connected socket
- Outputs/Return: status code; local IP string via pointer parameter
- Side effects: Memory allocation for IP string (not specified if caller must free)

## Control Flow Notes
This header establishes the abstraction layer. Actual implementations are in platform-specific .c files (`unix_socket.c`, `win_socket.c`). The `os.h` header uses preprocessor macros to alias these functions to platform-specific implementations at compile time. Typical lifecycle: socket setup ΓåÆ send/recv loop with optional select polling ΓåÆ socket close.

## External Dependencies
- `"os.h"` ΓÇô Platform detection and function aliasing macros
- **Unix**: `<fcntl.h>`, `<sys/socket.h>`, `<sys/select.h>`, `<netinet/in.h>`, `<arpa/inet.h>`, `<netdb.h>`, `<unistd.h>`, `<sys/time.h>`
- **Windows**: `<windows.h>`, `<winsock.h>`
- Standard C: `<stdlib.h>`, `<string.h>`, `<time.h>`
- Forward declarations: `struct sockaddr_in`, `struct hostent` (defined in system headers)
