# Source_Files/TCPMess/Message.cpp

## File Purpose
Implements message serialization and deserialization infrastructure for network transmission. Provides two concrete message handler classes: `SmallMessageHelper` (base for structured messages) and `BigChunkOfDataMessage` (for large binary payloads).

## Core Responsibilities
- Serialize (`deflate`) and deserialize (`inflate`) messages to/from wire format
- Manage memory for binary data buffers in `BigChunkOfDataMessage`
- Bridge between structured message types and big-endian stream serialization
- Support deep cloning and safe buffer copying

## Key Types / Data Structures
None (implementation file; all class definitions are in Message.h).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kSmallMessageBufferSize` | enum constant | static | Fixed 4 KiB buffer for small message serialization |

## Key Functions / Methods

### SmallMessageHelper::inflateFrom
- **Signature:** `bool inflateFrom(const UninflatedMessage& inUninflated)`
- **Purpose:** Deserialize a message from wire format by wrapping buffer in a big-endian stream and delegating to subclass.
- **Inputs:** `inUninflated` ΓÇö wire-format message wrapper
- **Outputs/Return:** `bool` ΓÇö success flag
- **Side effects:** Invokes subclass's `reallyInflateFrom()` to populate message state
- **Calls:** `AIStreamBE` constructor, `reallyInflateFrom()`
- **Notes:** Subclass must implement `reallyInflateFrom(AIStream&)`.

### SmallMessageHelper::deflate
- **Signature:** `UninflatedMessage* deflate() const`
- **Purpose:** Serialize message to wire format by delegating to subclass and wrapping in `UninflatedMessage`.
- **Inputs:** None (uses `this` state)
- **Outputs/Return:** `UninflatedMessage*` ΓÇö heap-allocated wire message (caller must `delete`)
- **Side effects:** Allocates 4 KiB buffer, writes serialized data, allocates result message
- **Calls:** `std::vector<byte>`, `AOStreamBE`, `reallyDeflateTo()`, `AOStream::tellp()`, `UninflatedMessage` constructor, `memcpy()`
- **Notes:** Fixed buffer size; subclass must implement `reallyDeflateTo(AOStream&)`.

### BigChunkOfDataMessage::BigChunkOfDataMessage (constructor)
- **Signature:** `BigChunkOfDataMessage(MessageTypeID inType, const byte* inBuffer, size_t inLength)`
- **Purpose:** Construct a message holding a large binary chunk.
- **Inputs:** `inType` ΓÇö message type ID; `inBuffer` ΓÇö byte buffer (or NULL); `inLength` ΓÇö buffer size
- **Outputs/Return:** None (constructor)
- **Side effects:** Calls `copyBufferFrom()` to allocate and copy buffer
- **Calls:** `copyBufferFrom()`
- **Notes:** Initializes `mLength=0` and `mBuffer=NULL` before copying; takes ownership of data.

### BigChunkOfDataMessage::inflateFrom
- **Signature:** `bool inflateFrom(const UninflatedMessage& inUninflated)`
- **Purpose:** Load message data from wire format.
- **Inputs:** `inUninflated` ΓÇö wire-format message
- **Outputs/Return:** `bool` ΓÇö always `true`
- **Side effects:** Calls `copyBufferFrom()` to replace current buffer
- **Calls:** `copyBufferFrom()`
- **Notes:** Simple buffer copy; does not validate message type.

### BigChunkOfDataMessage::deflate
- **Signature:** `UninflatedMessage* deflate() const`
- **Purpose:** Serialize to wire format.
- **Inputs:** None
- **Outputs/Return:** `UninflatedMessage*` ΓÇö heap-allocated (caller must `delete`)
- **Side effects:** Allocates new `UninflatedMessage` and copies buffer
- **Calls:** `UninflatedMessage` constructor, `memcpy()`
- **Notes:** Preserves type and buffer content exactly.

### BigChunkOfDataMessage::copyBufferFrom
- **Signature:** `void copyBufferFrom(const byte* inBuffer, size_t inLength)`
- **Purpose:** Safely replace internal buffer with a copy of new data.
- **Inputs:** `inBuffer` ΓÇö source buffer; `inLength` ΓÇö size in bytes
- **Outputs/Return:** None
- **Side effects:** Frees old buffer (`delete[]`), allocates new buffer (`new[]`), copies data
- **Calls:** `delete[]`, `new[]`, `memcpy()`
- **Notes:** Handles NULL/empty correctly; skips allocation if `inLength == 0`.

### BigChunkOfDataMessage::clone
- **Signature:** `Message* clone() const` (covariant return: `BigChunkOfDataMessage*`)
- **Purpose:** Deep copy this message.
- **Inputs:** None
- **Outputs/Return:** `BigChunkOfDataMessage*` ΓÇö new heap-allocated copy
- **Side effects:** Allocates new message with copied buffer
- **Calls:** `BigChunkOfDataMessage` constructor
- **Notes:** Uses constructor's copy logic.

### BigChunkOfDataMessage::~BigChunkOfDataMessage
- **Signature:** `~BigChunkOfDataMessage()`
- **Purpose:** Destructor; release heap-allocated buffer.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `delete[]` on `mBuffer`
- **Calls:** `delete[]`
- **Notes:** Handles NULL buffer safely.

## Control Flow Notes
This file implements a message serialization layer with no direct game-loop coupling visible. Messages flow through inflation/deflation:
- **Inflation:** Wire format (`UninflatedMessage`) ΓåÆ subclass's `inflateFrom()` ΓåÆ in-memory message state
- **Deflation:** In-memory message ΓåÆ subclass's `deflate()` ΓåÆ wire format (`UninflatedMessage`)

The design separates concerns: `SmallMessageHelper` (for templated structured messages) and `BigChunkOfDataMessage` (for raw binary chunks). Not inferable from this file whether these integrate with a network socket layer, message routing, or game state updates.

## External Dependencies
- **Standard library:** `<string.h>` (memcpy), `<vector>` (dynamic buffers)
- **Internal:** `Message.h` (class definitions, `UninflatedMessage`, forward decls), `AStream.h` (stream classes)
- **Stream classes used:** `AIStreamBE`, `AOStreamBE` (big-endian input/output, defined in AStream.h)
- **Macros:** `COVARIANT_RETURN` (enables covariant return types for virtual clone methods on older MSVC)
