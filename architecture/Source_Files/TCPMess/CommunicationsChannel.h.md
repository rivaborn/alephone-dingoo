# Source_Files/TCPMess/CommunicationsChannel.h

## File Purpose
Provides TCP-based network communication infrastructure for networked game state synchronization. Manages bidirectional message exchange with support for both synchronous and asynchronous receive patterns, automatic message serialization/deserialization, and connection lifecycle.

## Core Responsibilities
- Establish and manage TCP socket connections (connect, disconnect, peer address retrieval)
- Pump network I/O (separate inbound and outbound data movement from message dispatch)
- Dispatch incoming messages to handlers with optional type filtering
- Provide synchronous `receiveMessage()` and `receiveSpecificMessage()` with timeout semantics
- Queue outgoing messages and flush them to TCP with overall and inactivity timeouts
- Track last activity timestamps (`ticksAtLastReceive`, `ticksAtLastSend`)
- Support pluggable message handlers, message inflaters (deserializers), and arbitrary client state via Memento pattern
- Support factory-based acceptance of incoming connections

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Memento` | class | Abstract base; clients subclass to store arbitrary context on the channel (safe downcasting via `dynamic_cast<>()`) |
| `CommunicationsChannel` | class | Core channel; manages socket, message queues, headers, and lifecycle |
| `CommunicationsChannelFactory` | class | Listens on a port and creates `CommunicationsChannel` instances for accepted connections |
| `CommunicationResult` (private enum) | enum | Internal state: `kIncomplete`, `kComplete`, `kError` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kSSRAnyDataTimeout` | Uint32 enum | class constant | 10 seconds; max idle before abandoning a receive attempt |
| `kSSRAnyMessageTimeout` | Uint32 enum | class constant | 30 seconds; max overall time to receive any message |
| `kSSRSpecificMessageTimeout` | Uint32 enum | class constant | 30 seconds; max overall time to receive a specific message type |
| `kOutgoingOverallTimeout` | Uint32 enum | class constant | 10 minutes; max time to flush all outgoing messages |
| `kOutgoingInactivityTimeout` | Uint32 enum | class constant | 10 seconds; max idle time during flush |
| `kHeaderPackedSize` | int enum | class constant | 8 bytes; serialized message header size |
| `kHeaderMagic` | Uint32 enum | class constant | 0xDEAD; magic number in message headers |

## Key Functions / Methods

### pump
- **Signature:** `void pump()`
- **Purpose:** Move raw data between TCP socket and internal buffers; does not invoke message handlers.
- **Inputs:** None (uses internal socket and buffers)
- **Outputs/Return:** None
- **Side effects:** Reads/writes socket; updates `mTicksAtLastReceive` and `mTicksAtLastSend` on activity; may update incoming/outgoing header/message positions
- **Calls:** `pumpReceivingSide()`, `pumpSendingSide()`
- **Notes:** Non-blocking or low-latency; must be called regularly in main loop to maintain throughput

### dispatchOneIncomingMessage
- **Signature:** `bool dispatchOneIncomingMessage()`
- **Purpose:** Process one queued incoming message; invoke `MessageHandler` callback if set.
- **Inputs:** None (uses internal `mIncomingMessages` queue)
- **Outputs/Return:** `false` if queue empty; `true` if message was dispatched
- **Side effects:** Removes message from queue; calls handler; may delete message
- **Calls:** Message handler callback (if set)
- **Notes:** Returns immediately; non-blocking

### dispatchIncomingMessages
- **Signature:** `void dispatchIncomingMessages()`
- **Purpose:** Drain entire queue of incoming messages, dispatching each.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Processes all queued messages; calls handler for each
- **Calls:** `dispatchOneIncomingMessage()` in a loop

### receiveMessage (blocking)
- **Signature:** `Message* receiveMessage(Uint32 inOverallTimeout = kSSRAnyMessageTimeout, Uint32 inInactivityTimeout = kSSRAnyDataTimeout)`
- **Purpose:** Block until any message is fully received, timeout expires, or socket disconnects.
- **Inputs:** `inOverallTimeout` (ms, default 30s), `inInactivityTimeout` (ms, default 10s)
- **Outputs/Return:** Pointer to inflated `Message` (caller must delete); `NULL` on timeout or disconnect
- **Side effects:** May call pump, dispatch handlers; blocks
- **Calls:** `pump()`, `dispatchIncomingMessages()`
- **Notes:** Caller owns returned pointer; unrelated messages received are queued and dispatched via handlers

### receiveSpecificMessage (templated variants)
- **Signature:** 
  - `Message* receiveSpecificMessage(MessageTypeID inType, Uint32 inOverallTimeout, Uint32 inInactivityTimeout)`
  - `template <typename tMessage> tMessage* receiveSpecificMessage(MessageTypeID inType, ...)`
  - `template <typename tMessage> tMessage* receiveSpecificMessage(...)` (deduces type from template arg)
  - `template <typename tMessage> tMessage* receiveSpecificMessageOrThrow(...)`
- **Purpose:** Receive a message of a specific type with type-safe casting; unmatched messages are dispatched normally.
- **Inputs:** Message type ID (or inferred from template); timeouts as above
- **Outputs/Return:** Downcast pointer to `tMessage*`; `NULL` if timeout/disconnect (or exception if `...OrThrow` variant)
- **Side effects:** Blocking; may dispatch unmatched messages
- **Calls:** `receiveMessage()`, `dynamic_cast<tMessage*>()`
- **Notes:** Template variants provide type safety; `*OrThrow` raises `FailedToReceiveSpecificMessageException` on null result

### flushOutgoingMessages
- **Signature:** `void flushOutgoingMessages(bool dispatchIncomingMessages, Uint32 inOverallTimeout = kOutgoingOverallTimeout, Uint32 inInactivityTimeout = kOutgoingInactivityTimeout)`
- **Purpose:** Block until all queued outgoing messages are sent to TCP or timeout/disconnect occurs.
- **Inputs:** `dispatchIncomingMessages` (if true, invoke handlers on received data); timeouts
- **Outputs/Return:** None
- **Side effects:** Blocking; sends data; may dispatch incoming messages
- **Calls:** `pump()`, `dispatchIncomingMessages()` (conditional)
- **Notes:** Useful before waiting for response; single-channel version

### multipleFlushOutgoingMessages (static)
- **Signature:** `static void multipleFlushOutgoingMessages(std::vector<CommunicationsChannel*>&, bool dispatchIncomingMessages, Uint32 inOverallTimeout, Uint32 inInactivityTimeout)`
- **Purpose:** Efficiently flush multiple channels in one call (avoid repeated polls across many channels).
- **Inputs:** Vector of channels, dispatch flag, timeouts
- **Outputs/Return:** None
- **Side effects:** Blocking; sends data on all channels; may dispatch incoming messages
- **Notes:** Batch operation; typically used in server loop with many client channels

### enqueueOutgoingMessage
- **Signature:** `void enqueueOutgoingMessage(const Message& inMessage)`
- **Purpose:** Copy and queue a message for transmission.
- **Inputs:** Reference to message (copied internally)
- **Outputs/Return:** None
- **Side effects:** Deflates message and queues `UninflatedMessage*` in `mOutgoingMessages`
- **Calls:** `Message::deflate()`
- **Notes:** Caller retains ownership of input message

### connect
- **Signature:** 
  - `void connect(const std::string& inAddressString, Uint16 inPort)`
  - `void connect(const IPaddress& inAddress)`
- **Purpose:** Establish outbound TCP connection.
- **Inputs:** Address (string + port in host byte order, or preformed `IPaddress`)
- **Outputs/Return:** None
- **Side effects:** Opens socket; sets `mConnected`; resets buffers and positions
- **Calls:** SDL_net socket functions
- **Notes:** Documentation states failures cause disconnect; address string may be hostname or IP

### disconnect
- **Signature:** `void disconnect()`
- **Purpose:** Close connection and clean up socket.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Closes socket; sets `mConnected = false`

### peerAddress
- **Signature:** `IPaddress peerAddress() const`
- **Purpose:** Query the remote endpoint address.
- **Inputs:** None
- **Outputs/Return:** `IPaddress` struct (port in network byte order)
- **Calls:** SDL_net query function

### Activity tracking
- **Signature:** `Uint32 ticksAtLastReceive() const`, `Uint32 ticksAtLastSend() const`, `Uint32 millisecondsSinceLastReceive() const`, `Uint32 millisecondsSinceLastSend() const`
- **Purpose:** Gauge activity for timeout or monitoring logic.
- **Inputs:** None
- **Outputs/Return:** Tick count or milliseconds elapsed
- **Calls:** `SDL_GetTicks()` (for elapsed variants)
- **Notes:** Updated by `pump()` on any data movement; useful for heartbeat/watchdog logic

## Control Flow Notes
This class is typically driven by a game loop:
1. **Per-frame (or per-tick):** Call `pump()` to move network I/O
2. **Dispatch phase:** Call `dispatchIncomingMessages()` to invoke handlers on newly arrived data
3. **Blocking operations:** `receiveMessage()` / `flushOutgoingMessages()` used at synchronization points (e.g., waiting for a reply before proceeding)
4. **Factory pattern:** `CommunicationsChannelFactory` is created at startup, accepted connections yield new `CommunicationsChannel` instances for each client

The separation of pump (I/O movement) from dispatch (handler invocation) allows efficient polling of multiple channels in one pump loop before processing messages.

## External Dependencies
- **SDL_net:** `TCPsocket`, `IPaddress`, `SDL_GetTicks()`, `Uint8`, `Uint16`, `Uint32`
- **Message.h:** `Message`, `UninflatedMessage`, `MessageTypeID` (Uint16), `MessageInflater`, `MessageHandler` (forward declared)
- **Standard library:** `<list>`, `<string>`, `<memory>` (auto_ptr), `<stdexcept>`, `<vector>`
- **config.h:** Provides `DISABLE_NETWORKING` feature gate and platform defines
