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

## External Dependencies
- **SDL_net:** SDLNet_TCP_Open, SDLNet_TCP_Send, SDLNet_TCP_Recv, SDLNet_TCP_Close, SDLNet_ResolveHost, SDLNet_AllocSocketSet, SDLNet_TCP_AddSocket, SDLNet_CheckSockets, SDLNet_TCP_Accept, SDLNet_FreeSocketSet, SDLNet_TCP_GetPeerAddress
- **SDL:** SDL_GetTicks, SDL_Delay, SDL_SwapBE16
- **Platform-specific:** winsock2.h (WSAGetLastError, ioctlsocket), OpenTransport.h (macOS: OTSetNonBlocking, OTRcv, OTLook, OTRcvConnect, OTRcvOrderlyDisconnect, OTRcvDisconnect, OTRcvUDErr, OTEnterNotifier, OTLeaveNotifier), fcntl.h (Unix)
- **AStream.h:** AIStreamBE, AOStreamBE (serialization)
- **MessageInflater.h, MessageHandler.h:** Message inflation/deflation and callback handling
- **Message.h:** Message, UninflatedMessage base types (defined elsewhere)

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

## External Dependencies
- **SDL_net:** `TCPsocket`, `IPaddress`, `SDL_GetTicks()`, `Uint8`, `Uint16`, `Uint32`
- **Message.h:** `Message`, `UninflatedMessage`, `MessageTypeID` (Uint16), `MessageInflater`, `MessageHandler` (forward declared)
- **Standard library:** `<list>`, `<string>`, `<memory>` (auto_ptr), `<stdexcept>`, `<vector>`
- **config.h:** Provides `DISABLE_NETWORKING` feature gate and platform defines

# Source_Files/TCPMess/Message.cpp
## File Purpose
Implements message serialization and deserialization infrastructure for network transmission. Provides two concrete message handler classes: `SmallMessageHelper` (base for structured messages) and `BigChunkOfDataMessage` (for large binary payloads).

## Core Responsibilities
- Serialize (`deflate`) and deserialize (`inflate`) messages to/from wire format
- Manage memory for binary data buffers in `BigChunkOfDataMessage`
- Bridge between structured message types and big-endian stream serialization
- Support deep cloning and safe buffer copying

## External Dependencies
- **Standard library:** `<string.h>` (memcpy), `<vector>` (dynamic buffers)
- **Internal:** `Message.h` (class definitions, `UninflatedMessage`, forward decls), `AStream.h` (stream classes)
- **Stream classes used:** `AIStreamBE`, `AOStreamBE` (big-endian input/output, defined in AStream.h)
- **Macros:** `COVARIANT_RETURN` (enables covariant return types for virtual clone methods on older MSVC)

# Source_Files/TCPMess/Message.h
## File Purpose
Defines a polymorphic message abstraction layer for TCP network communication in Aleph One (Marathon-like game engine). Provides interfaces and concrete implementations for serializing/deserializing typed messages to/from raw byte buffers, supporting messages of varying sizes and complexity.

## Core Responsibilities
- Define `Message` base interface for inflation (deserialization) and deflation (serialization)
- Provide `UninflatedMessage` as a raw byte buffer wrapper for transmission/reception
- Offer `SmallMessageHelper` as a base for stream-based serializable messages
- Provide `BigChunkOfDataMessage` for large binary payloads
- Supply `SimpleMessage<T>` template for single-value typed messages
- Supply `DatalessMessage<T>` template for empty/flag messages (type-only, no data)

## External Dependencies
- `SDL.h` ΓÇô Uint8, Uint16 types
- `config.h` ΓÇô DISABLE_NETWORKING flag (entire file is guarded)
- `AIStream`, `AOStream` ΓÇô forward-declared; stream classes for serialization (defined elsewhere, likely in network module)

# Source_Files/TCPMess/MessageDispatcher.cpp
## File Purpose
Conditional compilation wrapper for the MessageDispatcher class. When `DISABLE_NETWORKING` is not defined, includes the header to enable message dispatching over TCP. The actual implementation is in the paired header file.

## Core Responsibilities
- Conditionally includes MessageDispatcher.h only when networking is enabled
- Acts as the compilation unit for the message dispatcher subsystem

## External Dependencies
- **config.h** ΓÇö autoconf-generated configuration header; checks `DISABLE_NETWORKING` to conditionally enable this module
- **MessageDispatcher.h** ΓÇö contains the class definition (inherits from `MessageHandler`, uses `std::map` for routing)
- **Message.h** / **MessageHandler.h** ΓÇö defined elsewhere; provide message types and handler interface

**Notes:**  
The .cpp file is minimal; nearly all logic is in the header. This is an early-2000s codebase (2003 copyright) where template-heavy designs favored header-only patterns.

# Source_Files/TCPMess/MessageDispatcher.h
## File Purpose
Implements a message routing system that dispatches incoming messages to type-specific handlers. Acts as a strategy selector and composite handler for a networking message system, enabling decoupled message processing based on message type.

## Core Responsibilities
- Register and unregister message handlers by type ID
- Route incoming messages to appropriate handlers based on type
- Provide default handler fallback for unregistered message types
- Support dynamic handler lookup and handler chain composition
- Inherit from `MessageHandler` to act as a transparent handler in message processing pipelines

## External Dependencies
- `#include <map>` ΓÇö C++ STL for `std::map`
- `#include "Message.h"` ΓÇö defines `Message`, `MessageTypeID` (Uint16)
- `#include "MessageHandler.h"` ΓÇö defines `MessageHandler` interface
- `CommunicationsChannel` ΓÇö forward declared; used as opaque channel context
- Guarded by `#if !defined(DISABLE_NETWORKING)` ΓÇö optional networking subsystem

# Source_Files/TCPMess/MessageHandler.cpp
## File Purpose
This is a compilation unit for the message handling subsystem of a networked game engine. The actual implementation resides in the header file (MessageHandler.h) due to C++ template requirements; this .cpp file provides no additional implementation beyond including the header.

## Core Responsibilities
- Acts as a compilation unit for the TCPMess (TCP messaging) subsystem
- Guards message handling code against builds with `DISABLE_NETWORKING` preprocessor flag
- Provides header-only template-based message handler adapters (actual logic defined in MessageHandler.h)

## External Dependencies
- `config.h` ΓÇö Provides `DISABLE_NETWORKING` macro and build configuration
- `MessageHandler.h` ΓÇö Contains all class and template definitions (forward-declared types: `Message`, `CommunicationsChannel`)
- `<cstdlib>` ΓÇö Standard library (included in header)

**Note:** This .cpp file is essentially a null implementation unit; all functionality is template-based and defined in the header. The file exists to ensure the compilation unit is present for linker purposes and to guard the module behind the `DISABLE_NETWORKING` flag.

# Source_Files/TCPMess/MessageHandler.h
## File Purpose
Defines a polymorphic message handler interface and template-based adapters for type-safe network message dispatch. Enables registering and invoking both function pointers and member methods as handlers for incoming messages on communication channels.

## Core Responsibilities
- Define abstract `MessageHandler` base class for polymorphic message handling
- Provide `TypedMessageHandlerFunction` template to wrap typed function pointers
- Provide `MessageHandlerMethod` template to wrap typed member method pointers  
- Enable type-safe dispatch of messages through dynamic casting and template specialization
- Provide `newMessageHandlerMethod()` factory function for convenient template instantiation

## External Dependencies
- Forward declarations: `Message`, `CommunicationsChannel` (defined elsewhere in networking subsystem)
- Standard library: `<cstdlib>` (for general heap operations, though explicit allocation only in factory)
- Conditional compilation: guarded by `#if !defined(DISABLE_NETWORKING)` from config.h
- No external dependencies beyond standard library

# Source_Files/TCPMess/MessageInflater.cpp
## File Purpose
Implements message deserialization (inflation) for the TCP messaging system. Uses a prototype-based factory pattern to reconstruct network messages from their wire format, with fallback to returning uninflated messages on errors.

## Core Responsibilities
- Deserialize "uninflated" (serialized) network messages into concrete `Message` objects
- Maintain a registry of message type prototypes indexed by `MessageTypeID`
- Clone and initialize message prototypes during deserialization
- Register and unregister message type prototypes
- Handle errors gracefully with fallback to uninflated messages
- Clean up prototype storage on destruction

## External Dependencies
- **Message.h** ΓÇô defines `Message` class (methods: `clone()`, `inflateFrom()`, `type()`) and `UninflatedMessage` (methods: `clone()`, `inflatedType()`)
- **Logging.h** ΓÇô logging macros (`logWarning1`, `logAnomaly1`)
- **config.h** ΓÇô build configuration (entire file gated by `!DISABLE_NETWORKING`)
- **`<map>`** ΓÇô STL container for prototype registry

# Source_Files/TCPMess/MessageInflater.h
## File Purpose
Defines `MessageInflater`, a message deserialization engine that reconstructs typed `Message` objects from wire-format `UninflatedMessage` containers. Uses the Prototype pattern to registry message types and dynamically instantiate the correct subclass during inflation.

## Core Responsibilities
- Deserialize raw message bytes into type-specific `Message` subclass instances
- Maintain a registry of known message type prototypes indexed by `MessageTypeID`
- Support runtime registration and deregistration of message prototypes
- Route incoming messages to appropriate inflater implementations via prototype cloning

## External Dependencies
- `#include "Message.h"` ΓÇô `Message`, `UninflatedMessage`, `MessageTypeID`
- `#include <map>` ΓÇô standard library container
- `#include "config.h"` ΓÇô build configuration; guarded by `#if !defined(DISABLE_NETWORKING)`
- `std::map` ΓÇô STL associative container



