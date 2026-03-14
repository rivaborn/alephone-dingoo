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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Message` | abstract class | Polymorphic base for all message types; defines inflate/deflate contract |
| `UninflatedMessage` | concrete class | Raw byte buffer container; wraps binary message data with type ID and length |
| `SmallMessageHelper` | abstract class | Base for messages using stream-based serialization (AIStream/AOStream) |
| `BigChunkOfDataMessage` | concrete class | Manages large binary payloads with buffer ownership semantics |
| `SimpleMessage<tValueType>` | template class | Generic single-value message; delegates to stream operators |
| `DatalessMessage<tMessageType>` | template class | Type-only message (no payload); validates empty inflation |
| `MessageTypeID` | typedef | `Uint16`; identifies message type |

## Global / File-Static State
None (COVARIANT_RETURN is a preprocessor macro for return-type covariance).

## Key Functions / Methods

### Message (abstract base)

**type()**
- Signature: `virtual MessageTypeID type() const = 0`
- Purpose: Return the type identifier for this message
- Inputs: None
- Outputs/Return: `MessageTypeID` (Uint16)
- Side effects: None
- Calls: None
- Notes: Pure virtual; must be overridden by all derived classes

**inflateFrom()**
- Signature: `virtual bool inflateFrom(const UninflatedMessage& inUninflated) = 0`
- Purpose: Deserialize message state from raw byte buffer
- Inputs: `UninflatedMessage&` reference containing serialized data
- Outputs/Return: `bool` (success/failure)
- Side effects: Modifies internal message state
- Calls: None (subclass-dependent)
- Notes: May raise exceptions on validation failure; pure virtual

**deflate()**
- Signature: `virtual UninflatedMessage* deflate() const = 0`
- Purpose: Serialize message to raw byte buffer for transmission
- Inputs: None
- Outputs/Return: Newly allocated `UninflatedMessage*` (caller must delete)
- Side effects: Heap allocation
- Calls: None (subclass-dependent)
- Notes: Pure virtual; caller owns returned pointer

**clone()**
- Signature: `virtual Message* clone() const = 0`
- Purpose: Create a deep copy of this message
- Inputs: None
- Outputs/Return: Newly allocated `Message*` (caller must delete)
- Side effects: Heap allocation
- Calls: None (subclass-dependent)
- Notes: Pure virtual; uses covariant return type (COVARIANT_RETURN macro)

---

### UninflatedMessage (concrete)

**Constructor (with optional buffer)**
- Signature: `UninflatedMessage(MessageTypeID inType, size_t inLength, Uint8* inBytes = NULL)`
- Purpose: Wrap or create a byte buffer with metadata
- Inputs: `inType` (message type ID), `inLength` (buffer size), `inBytes` (optional existing buffer)
- Outputs/Return: Object initialization
- Side effects: If `inBytes == NULL`, allocates new buffer; otherwise takes ownership
- Calls: `new Uint8[mLength]` (conditional)
- Notes: Ownership transfer: if `inBytes` provided, this object assumes responsibility for deallocation

**copyToThis()**
- Signature: `void copyToThis(const UninflatedMessage& inSource)` (private)
- Purpose: Deep-copy state from another UninflatedMessage
- Inputs: Source UninflatedMessage reference
- Outputs/Return: None
- Side effects: Allocates new buffer; copies metadata and data
- Calls: `new Uint8[mLength]`, `memcpy()`
- Notes: Used by copy constructor and assignment operator

**buffer()**
- Signature: `Uint8* buffer()` and `const Uint8* buffer() const`
- Purpose: Access raw byte data for reading/writing
- Inputs: None
- Outputs/Return: Pointer to internal buffer (mutable or const)
- Side effects: None
- Calls: None
- Notes: Clients write directly into mutable buffer; const overload for read-only access

---

### SmallMessageHelper (abstract, extends Message)

**inflateFrom()**
- Signature: `bool inflateFrom(const UninflatedMessage& inUninflated)` (non-pure, defined here)
- Purpose: Extract message state from byte buffer via stream deserialization
- Inputs: UninflatedMessage reference
- Outputs/Return: `bool` (success)
- Side effects: Calls reallyInflateFrom(AIStream&), which modifies internal state
- Calls: Creates AIStream, delegates to virtual `reallyInflateFrom()`
- Notes: Implementation detail; subclasses override `reallyInflateFrom()` instead

**deflate()**
- Signature: `UninflatedMessage* deflate() const`
- Purpose: Serialize message to byte buffer via stream
- Inputs: None
- Outputs/Return: Newly allocated UninflatedMessage*
- Side effects: Heap allocation; calls reallyDeflateTo(AOStream&)
- Calls: Creates AOStream, delegates to virtual `reallyDeflateTo()`
- Notes: Implementation detail; subclasses override `reallyDeflateTo()` instead

---

### BigChunkOfDataMessage (concrete)

**Constructor**
- Signature: `BigChunkOfDataMessage(MessageTypeID inType, const Uint8* inBuffer = NULL, size_t inLength = 0)`
- Purpose: Create a message wrapping a large binary payload
- Inputs: `inType` (message type), `inBuffer` (optional data pointer), `inLength` (data size)
- Outputs/Return: Object initialization
- Side effects: Heap allocation via `copyBufferFrom()` if buffer provided
- Calls: `copyBufferFrom()`
- Notes: Manages buffer lifetime; copy constructor initializes mLength and mBuffer to null first

**copyBufferFrom()**
- Signature: `void copyBufferFrom(const Uint8* inBuffer, size_t inLength)`
- Purpose: Replace internal buffer with a copy of provided data
- Inputs: Data pointer and size
- Outputs/Return: None
- Side effects: Deallocates old buffer; allocates and populates new one
- Calls: `memcpy()`
- Notes: Handles NULL input gracefully

**buffer()**
- Signature: `Uint8* buffer()` and `const Uint8* buffer() const`
- Purpose: Access raw payload bytes
- Inputs: None
- Outputs/Return: Pointer to internal buffer
- Side effects: None
- Calls: None

---

### SimpleMessage<tValueType> (template, extends SmallMessageHelper)

**Constructor (default-initialized)**
- Signature: `SimpleMessage(MessageTypeID inType)`
- Purpose: Create message with default-constructed value
- Inputs: Message type ID
- Outputs/Return: Object initialization
- Side effects: Placement-new constructs tValueType at &mValue
- Calls: `new (static_cast<void*>(&mValue)) tValueType()`
- Notes: Uses placement new to invoke default constructor

**Constructor (with value)**
- Signature: `SimpleMessage(MessageTypeID inType, const tValueType& inValue)`
- Purpose: Create message with provided value
- Inputs: Message type ID and reference to value
- Outputs/Return: Object initialization
- Side effects: Copies value into mValue
- Calls: None
- Notes: Direct member initialization

**reallyDeflateTo()**
- Signature: `void reallyDeflateTo(AOStream& inStream) const` (protected)
- Purpose: Serialize the value to stream
- Inputs: Output stream reference
- Outputs/Return: None
- Side effects: Writes to stream
- Calls: `inStream << mValue` (stream operator)
- Notes: Assumes `operator<<` is defined for tValueType

**reallyInflateFrom()**
- Signature: `bool reallyInflateFrom(AIStream& inStream)` (protected)
- Purpose: Deserialize the value from stream
- Inputs: Input stream reference
- Outputs/Return: `bool` (always true on success)
- Side effects: Modifies mValue
- Calls: `inStream >> mValue` (stream operator)
- Notes: Assumes `operator>>` is defined for tValueType

---

### DatalessMessage<tMessageType> (template, extends Message)

**inflateFrom()**
- Signature: `bool inflateFrom(const UninflatedMessage& inUninflated)`
- Purpose: Validate that received message matches expected type and has no payload
- Inputs: UninflatedMessage reference
- Outputs/Return: `bool` (true if type matches and length == 0)
- Side effects: None
- Calls: `inUninflated.inflatedType()`, `inUninflated.length()`
- Notes: Assertion-like validation; returns false on mismatch

**deflate()**
- Signature: `UninflatedMessage* deflate() const`
- Purpose: Create empty message of correct type
- Inputs: None
- Outputs/Return: Newly allocated `UninflatedMessage(kType, 0)`
- Side effects: Heap allocation
- Calls: `UninflatedMessage` constructor

---

## Control Flow Notes
This is a **serialization framework** layered on top of raw networking (AIStream/AOStream). Typical message lifecycle:
1. **Creation**: `SimpleMessage<T>`, `BigChunkOfDataMessage`, or custom `Message` subclass instantiated
2. **Serialization**: `deflate()` ΓåÆ `UninflatedMessage` (byte buffer) ΓåÆ network transmission
3. **Deserialization**: Raw bytes wrapped in `UninflatedMessage` ΓåÆ `inflateFrom()` ΓåÆ typed message reconstructed
4. **SmallMessageHelper** delegates to stream-based serialization (useful for structured data)
5. **BigChunkOfDataMessage** handles raw binary data directly

No explicit frame/update/render lifecycle; this is purely a message transport abstraction.

## External Dependencies
- `SDL.h` ΓÇô Uint8, Uint16 types
- `config.h` ΓÇô DISABLE_NETWORKING flag (entire file is guarded)
- `AIStream`, `AOStream` ΓÇô forward-declared; stream classes for serialization (defined elsewhere, likely in network module)
