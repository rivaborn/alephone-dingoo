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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OsSocket` | opaque struct | OS-specific socket handle; implementation hidden from callers |

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Os_Socket_Udp_Setup
- Signature: `int LNat_Os_Socket_Udp_Setup(OsSocket ** s)`
- Purpose: Create and initialize a UDP socket
- Inputs: Pointer to socket pointer (output parameter)
- Outputs/Return: Integer status code
- Side effects: Allocates OS socket resource
- Calls: Redirects to `LNat_Unix_Socket_Udp_Setup` or `LNat_Win_Socket_Udp_Setup`
- Notes: Uses double pointer for output allocation

### LNat_Os_Socket_Udp_Send
- Signature: `int LNat_Os_Socket_Udp_Send(OsSocket * s, const char * host_addr, short int port, char * buf, int amt, int * amt_sent)`
- Purpose: Send UDP datagram to a remote host
- Inputs: Socket handle, destination IP, port, buffer, byte count
- Outputs/Return: Status code; `amt_sent` populated with bytes actually sent
- Side effects: Network I/O
- Calls: Redirects to `LNat_Unix_Socket_Udp_Send` or `LNat_Win_Socket_Udp_Send`

### LNat_Os_Socket_Udp_Recv
- Signature: `int LNat_Os_Socket_Udp_Recv(OsSocket * s, const char * host_addr, short int port, char * buf, int amt, int * amt_recv, int timeout_sec)`
- Purpose: Receive UDP datagram (blocking with timeout)
- Inputs: Socket, expected source IP, port, buffer, capacity, timeout in seconds
- Outputs/Return: Status; `amt_recv` populated with bytes received
- Side effects: Network I/O, may block
- Calls: Redirects to platform-specific implementation

### LNat_Os_Socket_Connect
- Signature: `int LNat_Os_Socket_Connect(OsSocket ** s, const char * host_addr, short int port, int timeout_sec)`
- Purpose: Establish TCP connection to remote host
- Inputs: Socket pointer, destination IP, port, connection timeout
- Outputs/Return: Status code; socket created and connected on success
- Side effects: Network I/O, allocates socket resource

### LNat_Os_Socket_Send / LNat_Os_Socket_Recv
- TCP equivalents of UDP send/recv; operate on established connection
- Return bytes sent/received via output parameters

### LNat_Os_Get_Local_Ip
- Signature: `int LNat_Os_Get_Local_Ip(OsSocket *, char ** local_ip)`
- Purpose: Query local IP bound to a connected socket
- Outputs/Return: Populates `local_ip` with dynamically allocated string

## Control Flow Notes
This is a header-only abstraction layer. Likely used during:
- **Initialization**: `LNat_Os_Socket_Udp_Setup` / `LNat_Os_Socket_Connect` to establish sockets
- **Network loop**: `Send` / `Recv` calls per frame or tick
- **Shutdown**: `LNat_Os_Socket_Udp_Close` / `LNat_Os_Socket_Close` to clean up resources

## External Dependencies
- No includes in this file (purely declarations)
- Implementation files (`os.c` and platform-specific variants) provide actual socket and network code
- Depends on platform headers (`winsock2.h` for Windows, `sys/socket.h` for Unix) in implementation
