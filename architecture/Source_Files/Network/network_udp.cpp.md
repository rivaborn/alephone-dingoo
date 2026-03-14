# Source_Files/Network/network_udp.cpp

## File Purpose
Implements UDP datagram networking for Aleph One, replacing the old AppleTalk DDP protocol. Provides cross-platform UDP socket management via SDL_net, with dedicated thread-based packet reception and a callback-driven packet handler model.

## Core Responsibilities
- Initialize and shutdown the UDP network module
- Open/close UDP sockets and bind to ports
- Spawn and manage a dedicated receiver thread with high priority
- Receive incoming UDP packets and dispatch them via registered packet handlers
- Create/dispose of DDP frame structures for outgoing packets
- Send frame data to remote addresses via UDP

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| UDPpacket | typedef (SDL_net) | Raw UDP packet buffer allocated by SDL_net |
| DDPPacketBuffer | struct | Wraps received UDP data with protocol type, source address, and datagram size for handler callback |
| DDPFrame | struct | Frame structure for outgoing packets, holds data buffer and associated socket |
| SDLNet_SocketSet | typedef (SDL_net) | Socket monitoring set used by receive thread |
| SDL_Thread | typedef (SDL) | Receive thread handle |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sUDPPacketBuffer | UDPpacket* | static | Allocated packet buffer for receiving incoming UDP datagrams |
| ddpPacketBuffer | DDPPacketBuffer | static | Converted DDP packet passed to the handler callback |
| sSocket | UDPsocket | static | The single open UDP socket for send/receive |
| sSocketSet | SDLNet_SocketSet | static | Socket monitoring set for the receive thread |
| sPacketHandler | PacketHandlerProcPtr | static | Registered callback function invoked when packets arrive |
| sReceivingThread | SDL_Thread* | static | Handle to the dedicated packet-receiving thread |
| sKeepListening | volatile bool | static | Shutdown flag for the receiving thread |

## Key Functions / Methods

### receive_thread_function
- **Signature:** `static int receive_thread_function(void*)`
- **Purpose:** Main loop for the dedicated receiver thread; continuously monitors the socket and dispatches incoming packets.
- **Inputs:** Unused `void*` parameter (SDL_Thread convention).
- **Outputs/Return:** Returns 0 on clean exit.
- **Side effects:** Calls `sPacketHandler` callback when packets arrive; acquires/releases `mytm_mutex` around handler invocation; logs drops if mutex unavailable.
- **Calls:** `SDLNet_CheckSockets()`, `SDLNet_UDP_Recv()`, `take_mytm_mutex()`, `release_mytm_mutex()`, `memcpy()`, `sPacketHandler()` (callback), `fdprintf()`.
- **Notes:** Loops with 1000ms timeout to allow clean shutdown on `sKeepListening = false`. Mutex protects handler call but not packet buffer access itself; comments suggest all uses are within handler scope, mitigating race risk.

### NetDDPOpenSocket
- **Signature:** `OSErr NetDDPOpenSocket(short *ioPortNumber, PacketHandlerProcPtr packetHandler)`
- **Purpose:** Opens a UDP socket on the specified port and starts the receiving thread.
- **Inputs:** Port number in network byte order; packet handler callback function pointer.
- **Outputs/Return:** Returns 0 on success, ΓêÆ1 on socket allocation/creation failure.
- **Side effects:** Allocates `sUDPPacketBuffer`; opens `sSocket`; creates `sSocketSet`; spawns `sReceivingThread` with boosted priority; sets global state (`sKeepListening`, `sPacketHandler`).
- **Calls:** `SDLNet_AllocPacket()`, `SDLNet_UDP_Open()` (with byte-swapped port), `SDLNet_AllocSocketSet()`, `SDLNet_UDP_AddSocket()`, `SDL_CreateThread()`, `BoostThreadPriority()`, `fdprintf()`.
- **Notes:** Converts port from network byte order (big-endian) to host byte order for SDL_net. Thread priority boost may fail (logged but non-fatal). Comment notes port should be returned in `*ioPortNumber` but currently is not implemented.

### NetDDPCloseSocket
- **Signature:** `OSErr NetDDPCloseSocket(short portNumber)`
- **Purpose:** Closes the UDP socket and shuts down the receiving thread.
- **Inputs:** Port number (currently ignored per API comment).
- **Outputs/Return:** Returns 0.
- **Side effects:** Sets `sKeepListening = false`, waits for `sReceivingThread` to exit, frees socket set, deallocates packet buffer, closes socket. All globals reset to NULL.
- **Calls:** `SDL_WaitThread()`, `SDLNet_FreeSocketSet()`, `SDLNet_FreePacket()`, `SDLNet_UDP_Close()`.
- **Notes:** Must be called from main thread (priority reduction side effect noted in comments). Thread cleanup is synchronous; no partial close state.

### NetDDPSendFrame
- **Signature:** `OSErr NetDDPSendFrame(DDPFramePtr frame, NetAddrBlock *address, short protocolType, short port)`
- **Purpose:** Send a frame to a remote address via the open UDP socket.
- **Inputs:** Frame with data and size, destination address, protocol type and port (latter two unused).
- **Outputs/Return:** Returns 0 on success (send accepted), ΓêÆ1 on send failure.
- **Side effects:** Modifies global `sUDPPacketBuffer` with frame data and address.
- **Calls:** `memcpy()`, `SDLNet_UDP_Send()`.
- **Notes:** Asserts `frame->data_size <= ddpMaxData`. Reuses global packet buffer; assumes frame data outlives the send call or is copied by SDL_net.

### NetDDPNewFrame / NetDDPDisposeFrame
- **Signatures:** `DDPFramePtr NetDDPNewFrame(void)` / `void NetDDPDisposeFrame(DDPFramePtr frame)`
- **Purpose:** Allocate and deallocate frame structures.
- **Inputs/Outputs:** Allocation returns frame pointer (or NULL on failure); disposal takes frame pointer (checked for NULL).
- **Side effects:** Malloc/free; frames initialized with `socket = sSocket`.
- **Calls:** `malloc()`, `memset()`, `free()`.
- **Notes:** Simple wrappers; frames are opaque to sender.

### NetDDPOpen / NetDDPClose
- **Signatures:** `OSErr NetDDPOpen(void)` / `OSErr NetDDPClose(void)`
- **Purpose:** Module-level initialization and shutdown stubs.
- **Outputs/Return:** Always return 0 (no-op).
- **Notes:** Placeholder functions; actual setup/teardown is in socket open/close.

## Control Flow Notes
**Initialization:** `NetDDPOpen()` ΓåÆ `NetDDPOpenSocket()` spawns a high-priority receiver thread that blocks on `SDLNet_CheckSockets()` with a 1s timeout.

**Frame/Update:** The receiver thread continuously polls the socket. When a packet arrives, it is converted from SDL format to DDP format, protected by mutex, and dispatched to the registered handler synchronously.

**Sending:** Frame allocation is decoupled from sending; sender calls `NetDDPSendFrame()` with a prepared frame, which directly uses the global packet buffer.

**Shutdown:** `NetDDPCloseSocket()` signals the receiver thread and waits for clean exit before releasing resources. This is a blocking synchronous operation.

## External Dependencies
- **SDL_net**: `UDPsocket`, `UDPpacket`, `SDLNet_*` functions (socket, packet, and set operations).
- **SDL**: `SDL_Thread`, `SDL_CreateThread()`, `SDL_WaitThread()`.
- **mytm.h**: `take_mytm_mutex()`, `release_mytm_mutex()` for thread-safe handler dispatch.
- **thread_priority_sdl.h**: `BoostThreadPriority()` for elevating receiver thread priority.
- **Defined elsewhere:** `PacketHandlerProcPtr` callback type, `DDPPacketBuffer` / `DDPFrame` structures, `NetAddrBlock` address type, `kPROTOCOL_TYPE` constant, `fdprintf()` logging function.
