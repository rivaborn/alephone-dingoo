# Subsystem Overview

## Purpose

Provides non-blocking TCP-based message networking infrastructure for synchronizing game state between connected clients. Handles message framing, serialization/deserialization, routing to type-specific handlers, and connection lifecycle management with timeout-aware buffering.

## Key Files

| File | Role |
|------|------|
| `CommunicationsChannel.cpp/h` | TCP socket lifecycle, non-blocking I/O pumping, message framing with magic-number validation, buffered send/receive queues |
| `Message.cpp/h` | Polymorphic message abstraction; concrete implementations (`SmallMessageHelper`, `BigChunkOfDataMessage`, `SimpleMessage<T>`, `DatalessMessage<T>`) for wire format serialization |
| `MessageInflater.cpp/h` | Prototype-based factory for deserializing `UninflatedMessage` byte buffers into typed `Message` subclasses |
| `MessageDispatcher.cpp/h` | Message router that dispatches incoming `Message` objects to type-specific handlers via `std::map` lookup |
| `MessageHandler.cpp/h` | Handler interface and template-based adapters (`TypedMessageHandlerFunction`, `MessageHandlerMethod`) for function pointers and member methods |

## Core Responsibilities

- Establish TCP socket connections (connect/disconnect) with non-blocking mode and platform-specific setup (Windows winsock2, macOS OpenTransport, Unix fcntl)
- Perform message framing: prepend magic number and length header; validate on reception
- Buffer partial I/O: state machineΓÇôdriven send/receive across multiple socket operations
- Manage paired inbound/outbound message queues with memento-based client state storage
- Deserialize wire-format bytes into type-specific `Message` subclasses using a registered prototype registry
- Route dispatched messages to handler callables (functions or member methods) by `MessageTypeID`
- Support synchronous `receiveMessage()` and `receiveSpecificMessage()` with overall and inactivity timeout semantics
- Track activity timestamps (`ticksAtLastReceive`, `ticksAtLastSend`) for timeout detection
- Accept incoming connections via factory pattern (server-side)
- Support message inflation/deflation for encoding custom payload types

## Key Interfaces & Data Flow

**Exposed to other subsystems:**
- `CommunicationsChannel`: connect/disconnect, pump inbound/outbound I/O, dispatch messages, send messages, synchronous receive
- `MessageHandler`: polymorphic callback interface invoked by dispatcher on message arrival
- `MessageInflater`: registry for message type prototypes; deserializes `UninflatedMessage` ΓåÆ `Message`

**Consumed from other subsystems:**
- `SDL_net`: socket creation, send/recv operations, socket set management, peer address lookup
- `SDL`: timestamp retrieval (`SDL_GetTicks`), byte-order swapping
- `AStream` classes: big-endian stream serialization/deserialization
- Platform APIs: winsock2, OpenTransport (macOS), fcntl (Unix)

## Runtime Role

Network messages arrive as TCP data ΓåÆ `CommunicationsChannel` pumps inbound data into framed message buffers ΓåÆ `MessageInflater` reconstructs typed `Message` objects from wire format ΓåÆ `MessageDispatcher` routes by type ID to registered handlers ΓåÆ handlers execute as callbacks. Outbound messages queued via `CommunicationsChannel::sendMessage()` are serialized and flushed to TCP on demand or at frame boundaries. Timeout checks occur during receive operations and can interrupt waiting.

## Notable Implementation Details

- **State machine buffering:** Partial socket operations are resumed across pump calls; `state` field tracks read/write progress
- **Prototype factory pattern:** `MessageInflater` maintains a `std::map` of message type ΓåÆ prototype; deserialization clones and initializes prototypes rather than hardcoding subclass instantiation
- **Header-only templates:** `MessageDispatcher`, `MessageHandler`, and message templates (`SimpleMessage<T>`, `DatalessMessage<T>`) are defined primarily in headers to support template specialization
- **Memento pattern:** `MessageHandler*` and `Memento` (opaque client state) are stored per channel for stateful handler context
- **Optional subsystem:** All files guarded by `#if !defined(DISABLE_NETWORKING)` ΓÇö entire TCP messaging can be compiled out
- **Platform abstraction layer:** Socket operations differ between Windows (winsock2), macOS (OpenTransport), and Unix (BSD sockets); abstracted behind `#ifdef` blocks in `CommunicationsChannel.cpp`
