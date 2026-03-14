# Source_Files/Network/network_udp_opentransport.cpp

## File Purpose
Implements UDP networking for classic Mac OS via Apple's OpenTransport API, providing the DDP (Datagram Delivery Protocol) abstraction layer. Acts as a bridge between the game's network protocol (ring-based) and low-level UDP transport, using asynchronous notifications to handle incoming packets and a deferred-send queue to avoid interrupt-time operations.

## Core Responsibilities
- Initialize and teardown OpenTransport framework (`NetDDPOpen`, `NetDDPClose`)
- Create, bind, and configure UDP socket endpoints (`NetDDPOpenSocket`, `NetDDPCloseSocket`)
- Receive incoming UDP datagrams via asynchronous OpenTransport notifications
- Queue outgoing UDP frames for deferred transmission (`NetDDPSendFrame`, `NetDDPSendUnsentFrames`)
- Manage packet allocation and deallocation (`NetDDPNewFrame`, `NetDDPDisposeFrame`)
- Forward received packets to caller-supplied handler via callback
- Handle OpenTransport event notifications (data arrival, flow control, errors)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `DeferredSentFrame` | struct | Holds a queued outgoing UDP frame: data payload, size, destination address, and send flag |
| `DDPFrame` | struct (imported) | Outgoing frame container with data buffer and size |
| `DDPPacketBuffer` | struct (imported) | Received packet with source address, protocol type, and datagram data |
| `TUnitData` | typedef (OpenTransport) | OpenTransport unit data structure for UDP send/receive operations |
| `InetAddress` | typedef (OpenTransport) | Address structure (AF_INET, host, port, unused fields) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sEndpoint` | `EndpointRef` | static | OpenTransport UDP endpoint handle; `kOTInvalidEndpointRef` when closed |
| `sBoundAddress` | `InetAddress` | static | Local address successfully bound to the socket |
| `sIncomingAddress` | `InetAddress` | static | Source address of most recent incoming packet |
| `sIncomingUnitData` | `TUnitData` | static | Reusable buffer structure for receiving UDP datagrams |
| `sPacketHandler` | `PacketHandlerProcPtr` | static | Callback function invoked for each received packet; NULL if none installed |
| `sPacketBufferForPacketHandler` | `DDPPacketBuffer` | static | Staging buffer for validated received data before passing to handler |
| `sNotifierUPP` | `OTNotifyUPP` | static | Universal Procedure Pointer to async notification callback (`udpNotifier`) |
| `framesToSend` | `DeferredSentFrame[16]` | static | Array of queued outgoing frames; sent later from main loop |

## Key Functions / Methods

### NetDDPOpen
- **Signature:** `OSErr NetDDPOpen(void)`
- **Purpose:** Initialize OpenTransport framework; tolerates multiple calls (idempotent).
- **Inputs:** None
- **Outputs/Return:** `noErr` on success; error code (e.g., from `InitOpenTransport()`) on failure
- **Side effects:** Calls `InitOpenTransport()`, which patches `ExitToShell` to ensure `CloseOpenTransport()` is called at program exit
- **Calls:** `InitOpenTransport()`
- **Notes:** Handles `kEINVALErr` (returned if OT already initialized) as success. Assumes OT may already be active via SDL_net or SSLP.

### NetDDPClose
- **Signature:** `OSErr NetDDPClose(void)`
- **Purpose:** Cleanup hook for OpenTransport; currently a no-op.
- **Inputs:** None
- **Outputs/Return:** `noErr`
- **Side effects:** None
- **Calls:** None
- **Notes:** Placeholder; actual cleanup deferred to `InitOpenTransport`'s `ExitToShell` patch.

### NetDDPOpenSocket
- **Signature:** `OSErr NetDDPOpenSocket(short* ioPortNumber, PacketHandlerProcPtr inPacketHandler)`
- **Purpose:** Create, configure, and bind a UDP socket endpoint; install async notification handler.
- **Inputs:** `ioPortNumber` (desired port, in network byte order; 0 for OS-assigned); `inPacketHandler` (callback for received packets)
- **Outputs/Return:** `*ioPortNumber` set to actual bound port; returns `noErr` on success or error code on failure
- **Side effects:** Allocates `sIncomingUnitData.udata.buf` if not already allocated; modifies global state (`sEndpoint`, `sPacketHandler`, `sNotifierUPP`, `framesToSend[]`); installs OT notifier
- **Calls:** `OTCreateConfiguration()`, `OTOpenEndpoint()`, `OTSetBlocking()`, `OTSetAsynchronous()`, `OTBind()`, `OTInstallNotifier()`, `NewOTNotifyUPP()`, `handleDataArrival()` (to drain any pending packets), `logError1()`, `logNote()`
- **Notes:** 
  - Sets endpoint to synchronous nonblocking mode initially, then switches to asynchronous blocking.
  - Allocates receive buffer based on endpoint's TSDU (max datagram size).
  - Calls `handleDataArrival()` directly after installing notifier to catch any packets that arrived between bind and notifier installation.
  - Has cascading cleanup labels (`unbind_dealloc_close_and_return`, etc.) for proper error recovery.

### NetDDPCloseSocket
- **Signature:** `OSErr NetDDPCloseSocket(short socketNumber)`
- **Purpose:** Unbind and close the UDP endpoint; dealloc receive buffer; reset send queue.
- **Inputs:** `socketNumber` (ignored; only one socket supported)
- **Outputs/Return:** `noErr`
- **Side effects:** Calls `OTRemoveNotifier()`, `OTSetSynchronous()`, `OTSetNonBlocking()`, `OTUnbind()`, `OTCloseProvider()`; deallocates receive buffer; resets all `framesToSend[].send` flags to false
- **Calls:** `OTRemoveNotifier()`, `DisposeOTNotifyUPP()`, `OTSetSynchronous()`, `OTSetNonBlocking()`, `OTUnbind()`, `OTCloseProvider()`
- **Notes:** Guarded by check for `sEndpoint != kOTInvalidEndpointRef`. Socket number parameter unused.

### handleDataArrival
- **Signature:** `static void handleDataArrival(void)`
- **Purpose:** Drain incoming UDP datagrams from OpenTransport and dispatch to packet handler callback.
- **Inputs:** None (operates on global state)
- **Outputs/Return:** None
- **Side effects:** Calls `OTRcvUData()` in a loop; populates `sPacketBufferForPacketHandler`; invokes `sPacketHandler` callback for each valid packet; may log anomalies
- **Calls:** `OTRcvUData()`, `sPacketHandler()`, `memcpy()`, `logAnomaly1()` (conditional)
- **Notes:** 
  - Copies received data to `sPacketBufferForPacketHandler` rather than receiving directly to avoid garbage overwrite.
  - Asserts datagram size Γëñ `ddpMaxData` and address type = AF_INET.
  - Loops until `OTRcvUData()` returns `kOTNoDataErr`.
  - Handler is invoked only if datagram size > 0.

### udpNotifier (pascal callback)
- **Signature:** `static pascal void udpNotifier(void* inContext, OTEventCode inEventCode, OTResult inResult, void* cookie)`
- **Purpose:** Async OpenTransport notification callback; dispatches on event type.
- **Inputs:** `inContext` (unused), `inEventCode` (T_UDERR, T_GODATA, T_DATA), `inResult` (error result), `cookie` (unused)
- **Outputs/Return:** None
- **Side effects:** Calls `handleDataArrival()` on T_DATA; calls `OTRcvUDErr()` to clear error on T_UDERR; may log traces
- **Calls:** `handleDataArrival()`, `OTRcvUDErr()`, `logTrace2()` (conditional)
- **Notes:** 
  - Called at deferred task time (not main thread).
  - T_UDERR: clears error unconditionally even if not acted upon.
  - T_GODATA: flow-control lifted; unused in this implementation.
  - T_DATA: triggers `handleDataArrival()`.

### NetDDPNewFrame
- **Signature:** `DDPFramePtr NetDDPNewFrame(void)`
- **Purpose:** Allocate and zero-initialize a DDPFrame structure.
- **Inputs:** None
- **Outputs/Return:** Pointer to new `DDPFrame` or NULL if allocation fails
- **Side effects:** Dynamic memory allocation; calls `obj_clear()`
- **Calls:** `new`, `obj_clear()`
- **Notes:** None

### NetDDPDisposeFrame
- **Signature:** `void NetDDPDisposeFrame(DDPFramePtr inFrame)`
- **Purpose:** Deallocate a DDPFrame.
- **Inputs:** `inFrame` (pointer to frame to delete)
- **Outputs/Return:** None
- **Side effects:** Deallocates memory
- **Calls:** `delete`
- **Notes:** None

### NetDDPSendFrame
- **Signature:** `OSErr NetDDPSendFrame(DDPFramePtr inFrame, NetAddrBlock* inAddress, short inProtocolType, short inSocket)`
- **Purpose:** Queue a frame for transmission; actual sending deferred to `NetDDPSendUnsentFrames()`.
- **Inputs:** `inFrame` (data to send); `inAddress` (destination address); `inProtocolType` (protocolΓÇömust be `kPROTOCOL_TYPE`); `inSocket` (ignored)
- **Outputs/Return:** `noErr` (always, per comment: "the calling code doesn't look at this anyway")
- **Side effects:** Finds first available slot in `framesToSend[]` array and populates it; drops packet if all 16 slots full
- **Calls:** `memcpy()`
- **Notes:** 
  - Asserts `inProtocolType == kPROTOCOL_TYPE` and frame size Γëñ `ddpMaxData`.
  - Linear search for free slot; silently drops packet if queue full.
  - Copies frame data to deferred queue.

### NetDDPSendUnsentFrames
- **Signature:** `OSErr NetDDPSendUnsentFrames(void)`
- **Purpose:** Iterate queued frames and transmit via OpenTransport.
- **Inputs:** None
- **Outputs/Return:** `noErr` (always)
- **Side effects:** Calls `OTSndUData()` for each queued frame; resets send flags; may log notes
- **Calls:** `OTSndUData()`, `logNote1()`
- **Notes:** Iterates through all 16 slots and sends any with `send == true`. Errors logged but not propagated to caller.

## Control Flow Notes
**Initialization phase:**  
`NetDDPOpen()` ΓåÆ `NetDDPOpenSocket()` binds endpoint, installs notifier, and primes with any pending data.

**Runtime / main loop:**  
- Incoming: OpenTransport notifications trigger `udpNotifier()` ΓåÆ `handleDataArrival()` ΓåÆ invokes user callback.
- Outgoing: Game calls `NetDDPSendFrame()` to queue frames; main loop calls `NetDDPSendUnsentFrames()` to transmit deferred frames.

**Shutdown:**  
`NetDDPCloseSocket()` ΓåÆ `NetDDPClose()` (no-op).

**Deferred-send pattern rationale:**  
Since OpenTransport cannot send at interrupt time (deferred task), frames are queued in `framesToSend[]` and sent later from main loop context.

## External Dependencies
- **OpenTransport API:** `OTCreateConfiguration`, `OTOpenEndpoint`, `OTBind`, `OTSetBlocking`, `OTSetAsynchronous`, `OTInstallNotifier`, `OTRcvUData`, `OTSndUData`, `OTRcvUDErr`, `OTRemoveNotifier`, `OTUnbind`, `OTCloseProvider`, `OTSetSynchronous`, `OTSetNonBlocking`, `OTEnterNotifier`, `OTLeaveNotifier`, `NewOTNotifyUPP`, `DisposeOTNotifyUPP`, `InitOpenTransport` (Classic Mac OS only; guarded by `#ifndef __MACH__`)
- **SDL_net:** via `sdl_network.h` (comment notes "you don't even want to ask" why included)
- **SSLP:** via `network_private.h` (service location)
- **Logging:** `logError1()`, `logNote()`, `logAnomaly1()`, `logTrace2()` from `Logging.h`
- **Network internals:** `kPROTOCOL_TYPE`, `ddpMaxData` from `network_private.h` and `sdl_network.h`
- **Utilities:** `obj_clear()` (from cseries.h), `memcpy()` (standard C)
