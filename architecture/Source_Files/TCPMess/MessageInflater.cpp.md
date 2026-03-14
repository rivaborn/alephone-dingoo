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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MessageInflaterMap` | `typedef` | Maps `MessageTypeID` ΓåÆ `Message*` for prototype storage |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `mMap` | `MessageInflaterMap` | instance member | Stores cloned message prototypes keyed by type ID |

## Key Functions / Methods

### `inflate()`
- **Signature:** `Message* inflate(const UninflatedMessage& inSource)`
- **Purpose:** Deserialize an uninflated message into a fully constructed `Message` object
- **Inputs:** Reference to an `UninflatedMessage` containing serialized data and type ID
- **Outputs/Return:** Pointer to a new `Message` (or cloned `UninflatedMessage` on failure)
- **Side effects:** Allocates heap memory; logs warnings/anomalies on failure paths; calls `clone()` and `inflateFrom()` on message prototypes
- **Calls:** `mMap.find()`, `Message::clone()`, `Message::inflateFrom()`, `UninflatedMessage::clone()`, logging functions
- **Notes:** Implements multi-stage error recoveryΓÇöif prototype lookup, cloning, or deserialization fails, returns a clone of the original uninflated message as a fallback rather than NULL

### `learnPrototypeForType()`
- **Signature:** `void learnPrototypeForType(MessageTypeID inType, const Message& inPrototype)`
- **Purpose:** Register a message type by storing a clone of its prototype
- **Inputs:** Message type ID and reference to a prototype message
- **Outputs/Return:** None
- **Side effects:** Allocates heap memory; deletes any existing prototype for that type
- **Calls:** `Message::clone()`, `operator[]` on map, `delete`
- **Notes:** Replaces any previously registered prototype for the same type ID

### `removePrototypeForType()`
- **Signature:** `void removePrototypeForType(MessageTypeID inType)`
- **Purpose:** Unregister a message type and free its prototype
- **Inputs:** Message type ID to remove
- **Outputs/Return:** None
- **Side effects:** Deallocates heap memory; modifies map
- **Calls:** `mMap.find()`, `mMap.erase()`, `delete`

### `~MessageInflater()`
- **Signature:** (destructor)
- **Purpose:** Clean up all registered message prototypes
- **Side effects:** Deallocates all heap-allocated `Message*` pointers in the map
- **Notes:** Iterates and deletes all map entries; safe even if map is empty

## Control Flow Notes
Fits into the game's network initialization and runtime message handling:
1. **Init phase:** `learnPrototypeForType()` registers supported message types
2. **Runtime:** `inflate()` deserializes incoming network messages
3. **Shutdown:** Destructor cleans up all prototypes; `removePrototypeForType()` can deregister individual types

## External Dependencies
- **Message.h** ΓÇô defines `Message` class (methods: `clone()`, `inflateFrom()`, `type()`) and `UninflatedMessage` (methods: `clone()`, `inflatedType()`)
- **Logging.h** ΓÇô logging macros (`logWarning1`, `logAnomaly1`)
- **config.h** ΓÇô build configuration (entire file gated by `!DISABLE_NETWORKING`)
- **`<map>`** ΓÇô STL container for prototype registry
