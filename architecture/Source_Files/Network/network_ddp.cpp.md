# Source_Files/Network/network_ddp.cpp

## File Purpose

Implements DDP (Datagram Delivery Protocol) socket management for Classic Mac OS AppleTalk networking. Provides functions to open/close the AppleTalk driver, create/destroy sockets with packet handlers, allocate/release frames, and asynchronously send DDP packets to remote nodes.

## Core Responsibilities

- Driver lifecycle: open/close the .MPP (MacPack) AppleTalk driver
- Socket lifecycle: open DDP sockets with registered packet handlers, close sockets and clean up buffers
- Frame management: allocate and deallocate DDP frame structures
- Packet transmission: asynchronously send DDP frames with configurable protocol type and destination
- Listener integration: coordinate between C packet handler callbacks and 68k assembly socket listeners via UPPs (Universal Procedure Pointers)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `DDPFrame` | struct | Network frame with payload data, MPP parameter block, and WDS elements for scatter-gather transmission |
| `DDPPacketBuffer` | struct | Received packet buffer holding protocol type, source address, and datagram payload |
| `PacketHandlerUPP` | typedef | Callback signature for incoming DDP packets |
| `InitializeListenerUPP` | typedef | Callback to initialize the 68k socket listener with a packet handler |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ddpPacketBuffer` | `DDPPacketBufferPtr` | static | Current active packet buffer; `NULL` if no socket is open. Guards against multiple simultaneous listeners. |
| `listenerUPP` | `DDPSocketListenerUPP` | static | Listener UPP for the open socket (unused in current code). |
| `handlerUPP` (in `NetDDPOpenSocket`) | `PacketHandlerUPP` | static | Cached routine descriptor for the packet handler; created once and reused. |

## Key Functions / Methods

### NetDDPOpen
- **Signature:** `OSErr NetDDPOpen(void)`
- **Purpose:** Open the .MPP AppleTalk driver.
- **Inputs:** None.
- **Outputs/Return:** `OSErr` ΓÇö noErr on success.
- **Side effects:** Driver reference number stored in global `mppRefNum`.
- **Calls:** `OpenDriver()`
- **Notes:** Asserts that the driver reference number matches global state. Called during networking initialization.

### NetDDPClose
- **Signature:** `OSErr NetDDPClose(void)`
- **Purpose:** Close the .MPP driver (stub).
- **Inputs:** None.
- **Outputs/Return:** Always `noErr`.
- **Side effects:** None (function is a no-op).

### NetDDPOpenSocket
- **Signature:** `OSErr NetDDPOpenSocket(short *socketNumber, PacketHandlerProcPtr packetHandler)`
- **Purpose:** Open a DDP socket and register an incoming packet handler.
- **Inputs:** `socketNumber` (out), `packetHandler` callback.
- **Outputs/Return:** `OSErr`; socket number written to `*socketNumber` on success.
- **Side effects:** Allocates `ddpPacketBuffer`, creates/caches `handlerUPP`, calls MPP to open socket via `POpenSkt()`.
- **Calls:** `GetResource()`, `NewRoutineDescriptor()`, `CallUniversalProc()` (or direct call in 68k), `POpenSkt()`, `NewPtrClear()`, `MemError()`.
- **Notes:** Asserts no prior socket is open. Loads socket listener from a code resource, creates UPP for handler, calls listener initializer (68k assembly). On 68k, calls initializer directly; on PPC, uses `CallUniversalProc()`. Allocates `MPPParamBlock` on the stack.

### NetDDPCloseSocket
- **Signature:** `OSErr NetDDPCloseSocket(short socketNumber)`
- **Purpose:** Close a DDP socket and deallocate its packet buffer.
- **Inputs:** `socketNumber` to close.
- **Outputs/Return:** `OSErr`.
- **Side effects:** Calls `PCloseSkt()`, disposes `ddpPacketBuffer`, frees temp `MPPParamBlock`.
- **Calls:** `PCloseSkt()`, `DisposePtr()`, `MemError()`.

### NetDDPNewFrame
- **Signature:** `DDPFramePtr NetDDPNewFrame(void)`
- **Purpose:** Allocate a new DDP frame structure.
- **Inputs:** None.
- **Outputs/Return:** Pointer to allocated `DDPFrame`, or `NULL` on failure.
- **Side effects:** Allocates memory via `NewPtrClear()`.

### NetDDPDisposeFrame
- **Signature:** `void NetDDPDisposeFrame(DDPFramePtr frame)`
- **Purpose:** Deallocate a DDP frame.
- **Inputs:** Frame pointer.
- **Side effects:** Frees frame memory.
- **Calls:** `DisposePtr()`.

### NetDDPSendFrame
- **Signature:** `OSErr NetDDPSendFrame(DDPFramePtr frame, NetAddrBlock *address, short protocolType, short socket)`
- **Purpose:** Asynchronously send a DDP frame to a remote address.
- **Inputs:** Frame, destination address, protocol type, socket.
- **Outputs/Return:** `OSErr` from `PWriteDDP()`.
- **Side effects:** Writes frame header and data into MPP parameter block, invokes `PWriteDDP()` asynchronously. Static counter `count` tracks retries.
- **Calls:** `BuildDDPwds()`, `PWriteDDP()`, `dprintf()`.
- **Notes:** Checks prior `ioResult` for completion; retries if prior call was incomplete (tracks count). Ignores `asyncUncompleted` and re-allows the send. Asserts frame size Γëñ `ddpMaxData` (586 bytes).

## Control Flow Notes

- **Init:** Call `NetDDPOpen()` to open driver, then `NetDDPOpenSocket()` to open a socket with a packet handler callback.
- **Frame:** Allocate frames with `NetDDPNewFrame()`, populate data, send with `NetDDPSendFrame()`, dispose with `NetDDPDisposeFrame()`.
- **Shutdown:** Call `NetDDPCloseSocket()`, then `NetDDPClose()`.
- Packet reception is event-driven via the registered `packetHandler` callback, invoked by the 68k socket listener when DDP packets arrive.

## External Dependencies

- **Apple MacOS APIs:** `OpenDriver()`, `GetResource()`, `HLock()`, `HNoPurge()`, `StripAddress()`, `NewPtrClear()`, `MemError()`, `DisposePtr()`, `NewRoutineDescriptor()`, `CallUniversalProc()`, `GetCurrentISA()`.
- **AppleTalk/MPP macros/functions:** `POpenSkt()`, `PCloseSkt()`, `PWriteDDP()`, `BuildDDPwds()`.
- **Defined elsewhere:** `mppRefNum` (global), socket listener code resource (`SOCK`, ID 128), `DDPSocketListenerUPP` type.
