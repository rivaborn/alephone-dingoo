# Source_Files/TCPMess/CommunicationsChannel.cpp

## File Purpose
Implements TCP-based message networking for a game engine. Provides non-blocking socket communication with buffered message framing, header validation, timeout-aware receive/send operations, and batched message dispatch. Handles both client connections and server acceptance of incoming connections.

## Core Responsibilities
- TCP socket lifecycle management (connect, disconnect, non-blocking mode setup)
- Message framing with magic-number validation and length checking
- State machine-driven buffered send/receive with partial I/O handling
- Incoming and outgoing message queues with optional message inflation/deflation
- Blocking synchronous receive operations with overall and inactivity timeouts
- Message handler callbacks and memento storage per channel
- Batch flush coordination for multiple channels
- Server-side connection acceptance via factory pattern
- Platform-specific socket handling (Windows, macOS, Unix)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CommunicationsChannel | class | Single bidirectional TCP message connection |
| CommunicationsChannelFactory | class | Accepts incoming TCP connections on a listening socket |
| CommunicationResult | enum | Send/receive operation status (kIncomplete, kComplete, kError) |
| _TCPsocket (macOS) | struct | Internal SDL_net socket structure (macOS-specific override) |
| MessageQueue | typedef | std::list<Message*> for incoming messages |
| UninflatedMessageQueue | typedef | std::list<UninflatedMessage*> for outgoing messages |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| kMaximumMessageLength | enum (4MB) | static | Reject messages larger than this |
| kSSRPumpInterval | enum (50ms) | static | Polling interval for receive operations |
| kFlushPumpInterval | enum (50ms) | static | Polling interval for flush operations |
| kHeaderPackedSize | enum (8 bytes) | static | Serialized header size |
| kHeaderMagic | enum (0xDEAD) | static | Magic number for header validation |

## Key Functions / Methods

### receive_some
- **Signature:** `CommunicationResult receive_some(TCPsocket, byte*, size_t& ioPos, size_t len)`
- **Purpose:** Low-level non-blocking recv wrapper; updates position incrementally.
- **Inputs:** socket, buffer, current position, target length
- **Outputs/Return:** kIncomplete, kComplete, or kError; increments ioPos with bytes received
- **Side effects:** Calls SDLNet_TCP_Recv or platform-specific Our_TCP_Recv; disconnects on error; updates mTicksAtLastReceive
- **Calls:** SDLNet_TCP_Recv or Our_TCP_Recv (macOS), disconnect()
- **Notes:** Platform-specific error handling; EAGAIN/WSAEWOULDBLOCK treated as no-data (not error)

### send_some
- **Signature:** `CommunicationResult send_some(TCPsocket, byte*, size_t& ioPos, size_t len)`
- **Purpose:** Low-level non-blocking send wrapper; updates position incrementally.
- **Inputs:** socket, buffer, current position, target length
- **Outputs/Return:** kIncomplete, kComplete, or kError; increments ioPos with bytes sent
- **Side effects:** Calls SDLNet_TCP_Send; disconnects on error; updates mTicksAtLastSend
- **Calls:** SDLNet_TCP_Send, disconnect()

### receiveHeader / _receiveMessage / sendHeader / sendMessage
- **Purpose:** State machine methods for header/message framing in both directions.
- **Side effects:** Allocate/deallocate UninflatedMessage; manage position counters; enqueue/dequeue messages
- **Calls:** receive_some / send_some; AIStreamBE deserialize; mMessageInflater->inflate()

### pump / pumpReceivingSide / pumpSendingSide
- **Purpose:** Main I/O loop; drives state machine until blocked or error. Called repeatedly.
- **Side effects:** Sends pending messages, receives available data, transitions between header/message states
- **Calls:** receiveHeader, _receiveMessage, sendHeader, sendMessage

### receiveMessage / receiveSpecificMessage
- **Signature:** `Message* receiveMessage(Uint32 overallTimeout, Uint32 inactivityTimeout)`
- **Purpose:** Blocking receive with dual timeout semantics (overall deadline + inactivity threshold).
- **Inputs:** timeout in ms (overall and inactivity), message type filter (specific version)
- **Outputs/Return:** Pointer to inflated Message or NULL on timeout/disconnect
- **Side effects:** Repeated pump() + SDL_Delay(); dispatches non-matching messages (specific version)
- **Calls:** pump(), dispatchIncomingMessages()
- **Notes:** Caller owns returned pointer; inactivity timeout resets on each received byte

### flushOutgoingMessages / multipleFlushOutgoingMessages
- **Signature:** `void flushOutgoingMessages(bool dispatchIncoming, Uint32 overall, Uint32 inactivity)`
- **Purpose:** Block until all queued outgoing messages sent or timeout; optionally dispatch incoming.
- **Side effects:** Repeated pump() + SDL_Delay(); optional dispatchIncomingMessages()
- **Calls:** pump(), dispatchIncomingMessages()

### connect (2 overloads)
- **Signature:** `void connect(const IPaddress&)` / `void connect(const std::string&, uint16)`
- **Purpose:** Establish outgoing TCP connection; set non-blocking mode; reset buffers.
- **Inputs:** IP address or hostname + port
- **Side effects:** Allocates socket; resets incoming message state; clears message queues
- **Calls:** SDLNet_TCP_Open, SDLNet_ResolveHost, MakeTCPsocketNonBlocking()

### disconnect
- **Purpose:** Close socket and discard all pending messages.
- **Side effects:** Clears outgoing queue; resets state
- **Calls:** SDLNet_TCP_Close

### dispatchOneIncomingMessage / dispatchIncomingMessages
- **Purpose:** Invoke message handler callback and delete processed messages.
- **Side effects:** Calls mMessageHandler->handle(); pops from queue; deallocates message
- **Calls:** messageHandler()->handle()

### MakeTCPsocketNonBlocking
- **Signature:** `static void MakeTCPsocketNonBlocking(TCPsocket*)`
- **Purpose:** Platform-specific non-blocking socket setup via ioctlsocket (Win32), OTSetNonBlocking (macOS), or fcntl (Unix).
- **Side effects:** Modifies socket descriptor

### CommunicationsChannelFactory::newIncomingConnection
- **Purpose:** Non-blocking accept; returns new CommunicationsChannel for accepted socket.
- **Outputs/Return:** Pointer to new CommunicationsChannel or NULL if no pending connection
- **Side effects:** Allocates new channel; sets non-blocking mode
- **Calls:** SDLNet_AllocSocketSet, SDLNet_CheckSockets, SDLNet_TCP_Accept, MakeTCPsocketNonBlocking

## Control Flow Notes
**Pump-and-Dispatch Pattern:** This implements a typical game-engine I/O pattern: `pump()` moves raw bytes (non-blocking), then `dispatchIncomingMessages()` invokes callbacks. Synchronous recv methods (receiveMessage, flushOutgoingMessages) wrap pump loops with timeout logic.

**State Machine:** Receiving alternates between `receiveHeader()` ΓåÆ `_receiveMessage()` based on whether a full header is available. Similarly, sending alternates between `sendHeader()` and `sendMessage()`.

**Timeouts:** Dual-timeout model: *overall deadline* (hard limit on elapsed time) and *inactivity threshold* (max time with no new data received/sent). Inactivity resets on every I/O event.

## External Dependencies
- **SDL_net:** SDLNet_TCP_Open, SDLNet_TCP_Send, SDLNet_TCP_Recv, SDLNet_TCP_Close, SDLNet_ResolveHost, SDLNet_AllocSocketSet, SDLNet_TCP_AddSocket, SDLNet_CheckSockets, SDLNet_TCP_Accept, SDLNet_FreeSocketSet, SDLNet_TCP_GetPeerAddress
- **SDL:** SDL_GetTicks, SDL_Delay, SDL_SwapBE16
- **Platform-specific:** winsock2.h (WSAGetLastError, ioctlsocket), OpenTransport.h (macOS: OTSetNonBlocking, OTRcv, OTLook, OTRcvConnect, OTRcvOrderlyDisconnect, OTRcvDisconnect, OTRcvUDErr, OTEnterNotifier, OTLeaveNotifier), fcntl.h (Unix)
- **AStream.h:** AIStreamBE, AOStreamBE (serialization)
- **MessageInflater.h, MessageHandler.h:** Message inflation/deflation and callback handling
- **Message.h:** Message, UninflatedMessage base types (defined elsewhere)
