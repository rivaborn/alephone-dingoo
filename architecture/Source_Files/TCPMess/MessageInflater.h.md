# Source_Files/TCPMess/MessageInflater.h

## File Purpose
Defines `MessageInflater`, a message deserialization engine that reconstructs typed `Message` objects from wire-format `UninflatedMessage` containers. Uses the Prototype pattern to registry message types and dynamically instantiate the correct subclass during inflation.

## Core Responsibilities
- Deserialize raw message bytes into type-specific `Message` subclass instances
- Maintain a registry of known message type prototypes indexed by `MessageTypeID`
- Support runtime registration and deregistration of message prototypes
- Route incoming messages to appropriate inflater implementations via prototype cloning

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MessageInflater` | class | Main inflater; owns a map of message type prototypes |
| `MessageInflaterMap` | typedef | `std::map<MessageTypeID, Message*>` storing type ΓåÆ prototype bindings |
| `mMap` | member (private) | The actual prototype registry |

## Global / File-Static State
None.

## Key Functions / Methods

### inflate
- Signature: `Message* inflate(const UninflatedMessage& inSource)`
- Purpose: Deserialize a raw message into a typed `Message` object
- Inputs: `inSource` ΓÇô serialized message with type ID and buffer
- Outputs/Return: Heap-allocated `Message*` subclass instance (caller must `delete`)
- Side effects: Memory allocation; no state mutation
- Calls: Likely calls `clone()` on the prototype from `mMap[inSource.type()]`, then `inflateFrom()` on the clone
- Notes: Implementation not visible; must find the matching prototype, clone it, and populate with data. Behavior undefined if type not in registry (fatal or null return)

### learnPrototype
- Signature: `void learnPrototype(const Message& inPrototype)`
- Purpose: Register a message prototype by extracting its type ID
- Inputs: `inPrototype` ΓÇô a fully-constructed message instance (acts as a factory template)
- Outputs/Return: void
- Side effects: Inserts/overwrites entry in `mMap`
- Calls: `learnPrototypeForType(inPrototype.type(), inPrototype)`
- Notes: Convenience wrapper; delegates to `learnPrototypeForType`

### learnPrototypeForType
- Signature: `void learnPrototypeForType(MessageTypeID inType, const Message& inPrototype)`
- Purpose: Register a prototype under an explicit type ID (allows decoupling type from message instance)
- Inputs: `inType` ΓÇô the message type identifier; `inPrototype` ΓÇô the prototype instance
- Outputs/Return: void
- Side effects: Stores a copy/reference in `mMap[inType]`
- Calls: (not visible; likely calls `clone()` to store a copy)
- Notes: Overwrites any existing prototype for `inType`

### removePrototypeForType
- Signature: `void removePrototypeForType(MessageTypeID inType)`
- Purpose: Unregister a message type from the inflater
- Inputs: `inType` ΓÇô message type to remove
- Outputs/Return: void
- Side effects: Erases entry from `mMap`; may deallocate the prototype
- Calls: (not visible)
- Notes: Subsequent `inflate()` calls for this type will fail or return null

### ~MessageInflater (destructor)
- Signature: `~MessageInflater()`
- Purpose: Clean up all stored prototypes
- Side effects: Deallocates all `Message*` in `mMap`
- Notes: Implementation not visible; must iterate `mMap` and `delete` each entry

## Control Flow Notes
Typical flow: Early in initialization, subsystem learns prototypes for all known message types via `learnPrototype()`. Later, network code receives raw `UninflatedMessage` payloads and calls `inflate()` to reconstruct typed messages for dispatch. Prototypes are long-lived; cleanup occurs at shutdown via destructor.

## External Dependencies
- `#include "Message.h"` ΓÇô `Message`, `UninflatedMessage`, `MessageTypeID`
- `#include <map>` ΓÇô standard library container
- `#include "config.h"` ΓÇô build configuration; guarded by `#if !defined(DISABLE_NETWORKING)`
- `std::map` ΓÇô STL associative container

## Notes
- Memory management: Caller owns returned `Message*` from `inflate()`; `MessageInflater` owns stored prototypes
- The actual `.cpp` implementation is not provided; core logic in `inflate()` is deferred
- Type-safe prototype pattern: relies on virtual `clone()` and `inflateFrom()` in `Message` hierarchy
- No visible error handling in header; behavior on unknown type IDs not specified
