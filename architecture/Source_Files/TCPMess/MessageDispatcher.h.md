# Source_Files/TCPMess/MessageDispatcher.h

## File Purpose
Implements a message routing system that dispatches incoming messages to type-specific handlers. Acts as a strategy selector and composite handler for a networking message system, enabling decoupled message processing based on message type.

## Core Responsibilities
- Register and unregister message handlers by type ID
- Route incoming messages to appropriate handlers based on type
- Provide default handler fallback for unregistered message types
- Support dynamic handler lookup and handler chain composition
- Inherit from `MessageHandler` to act as a transparent handler in message processing pipelines

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MessageDispatcherMap` | typedef | Maps `MessageTypeID` ΓåÆ `MessageHandler*` for O(log n) type-based lookup |

## Global / File-Static State
None.

## Key Functions / Methods

### MessageDispatcher (constructor)
- Signature: `MessageDispatcher() : mDefaultHandler(NULL)`
- Purpose: Initialize dispatcher with empty handler map and no default handler
- Inputs: None
- Outputs/Return: New instance
- Side effects: None
- Calls: None
- Notes: Inline; null default indicates no fallback initially

### setHandlerForType
- Signature: `void setHandlerForType(MessageHandler* inHandler, MessageTypeID inType)`
- Purpose: Register or unregister a handler for a specific message type
- Inputs: `inHandler` (handler to register; NULL = unregister), `inType` (message type ID)
- Outputs/Return: None
- Side effects: Modifies `mMap` (insert or erase)
- Calls: `clearHandlerForType()` (if `inHandler == NULL`)
- Notes: NULL handler acts as unregister; otherwise inserts/overwrites in map

### handlerForType
- Signature: `MessageHandler* handlerForType(MessageTypeID inType)`
- Purpose: Look up handler for a type, with default fallback
- Inputs: `inType` (message type ID)
- Outputs/Return: Handler pointer (type-specific if found, else default; never NULL unless no default set)
- Side effects: None (read-only)
- Calls: None
- Notes: Returns `mDefaultHandler` if type not in map; can return NULL if no default

### handlerForTypeNoDefault
- Signature: `MessageHandler* handlerForTypeNoDefault(MessageTypeID inType)`
- Purpose: Look up handler for a type without fallback
- Inputs: `inType` (message type ID)
- Outputs/Return: Handler pointer or NULL
- Side effects: None (read-only)
- Calls: None
- Notes: Pure lookup; useful for type-strict routing

### clearHandlerForType
- Signature: `void clearHandlerForType(MessageTypeID inType)`
- Purpose: Unregister handler for a specific type
- Inputs: `inType` (message type ID)
- Outputs/Return: None
- Side effects: Erases entry from `mMap`
- Calls: None
- Notes: Safe even if type not found

### setDefaultHandler / defaultHandler
- Signature: `void setDefaultHandler(MessageHandler*)` / `MessageHandler* defaultHandler() const`
- Purpose: Set or retrieve fallback handler for unregistered types
- Inputs: Handler pointer (setter) / None (getter)
- Outputs/Return: None (setter) / current default (getter)
- Side effects: Setter modifies `mDefaultHandler`
- Calls: None
- Notes: Getter is const-correct

### handle (override)
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)`
- Purpose: Dispatch a message to its registered handler (implements `MessageHandler` interface)
- Inputs: `inMessage` (message to dispatch), `inChannel` (originating channel)
- Outputs/Return: None
- Side effects: Calls target handler's `handle()` method (message side effects delegated)
- Calls: `handlerForType()`, target handler's `handle()`
- Notes: Core dispatch logic; does nothing if handler is NULL

## Control Flow Notes
- **Message reception phase**: When a message arrives on a `CommunicationsChannel`, the dispatcher receives it via `handle()`.
- **Dispatch**: Looks up handler by message type; routes to registered handler or default.
- **Delegation**: Actual message processing happens in registered handler; dispatcher is transparent.
- Fits into a chain-of-responsibility or strategy pattern for decoupled message handling.

## External Dependencies
- `#include <map>` ΓÇö C++ STL for `std::map`
- `#include "Message.h"` ΓÇö defines `Message`, `MessageTypeID` (Uint16)
- `#include "MessageHandler.h"` ΓÇö defines `MessageHandler` interface
- `CommunicationsChannel` ΓÇö forward declared; used as opaque channel context
- Guarded by `#if !defined(DISABLE_NETWORKING)` ΓÇö optional networking subsystem
