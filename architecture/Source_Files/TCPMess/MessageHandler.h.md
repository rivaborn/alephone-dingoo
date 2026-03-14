# Source_Files/TCPMess/MessageHandler.h

## File Purpose
Defines a polymorphic message handler interface and template-based adapters for type-safe network message dispatch. Enables registering and invoking both function pointers and member methods as handlers for incoming messages on communication channels.

## Core Responsibilities
- Define abstract `MessageHandler` base class for polymorphic message handling
- Provide `TypedMessageHandlerFunction` template to wrap typed function pointers
- Provide `MessageHandlerMethod` template to wrap typed member method pointers  
- Enable type-safe dispatch of messages through dynamic casting and template specialization
- Provide `newMessageHandlerMethod()` factory function for convenient template instantiation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `MessageHandler` | Abstract class | Base interface for all message handlers; defines pure virtual `handle()` method |
| `TypedMessageHandlerFunction<tMessage, tChannel>` | Template class | Adapts a function pointer to the `MessageHandler` interface with type safety |
| `MessageHandlerMethod<tTargetClass, tMessage, tChannel>` | Template class | Adapts a member method pointer to the `MessageHandler` interface with type safety |
| `MessageHandlerFunction` | Typedef | Convenience alias for `TypedMessageHandlerFunction<Message>` (untyped base case) |

## Global / File-Static State
None.

## Key Functions / Methods

### MessageHandler::handle
- Signature: `virtual void handle(Message* inMessage, CommunicationsChannel* inChannel) = 0`
- Purpose: Pure virtual interface for processing an incoming message
- Inputs: `inMessage` (message to process), `inChannel` (source channel)
- Outputs/Return: void
- Side effects: None prescribed; implementations dictate behavior
- Calls: None (pure virtual)
- Notes: All concrete handlers must override; enables polymorphic invocation

### TypedMessageHandlerFunction::handle
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)` override
- Purpose: Delegates to wrapped function pointer after type-safe downcasting
- Inputs: Polymorphic base pointers (`Message*`, `CommunicationsChannel*`)
- Outputs/Return: void
- Side effects: Invokes wrapped function with casted pointers
- Calls: `dynamic_cast<tMessage*>()`, `dynamic_cast<tChannel*>()`, function pointer dereference
- Notes: Null-checks function pointer before invocation; casts may fail silently (yielding NULL)

### MessageHandlerMethod::handle
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)` override
- Purpose: Delegates to wrapped member method after type-safe downcasting
- Inputs: Polymorphic base pointers
- Outputs/Return: void
- Side effects: Invokes wrapped member method with casted pointers
- Calls: `dynamic_cast<tMessage*>()`, `dynamic_cast<tChannel*>()`, member method invocation
- Notes: Null-checks both object and method pointers; dynamic casts may fail

### newMessageHandlerMethod
- Signature: `static inline MessageHandlerMethod<...>* newMessageHandlerMethod(tTargetClass* targetObject, void (tTargetClass::*targetMethod)(...))`
- Purpose: Factory function to construct a typed handler with automatic template deduction
- Inputs: `targetObject` (object instance), `targetMethod` (member function pointer)
- Outputs/Return: Heap-allocated `MessageHandlerMethod` instance
- Side effects: Dynamic memory allocation
- Calls: `MessageHandlerMethod` constructor
- Notes: Inline for compile-time specialization; caller owns returned pointer

## Control Flow Notes
Defines handler abstraction; actual control flow depends on dispatcher. Likely usage: network message dispatcher receives polymorphic `Message*`, looks up registered handler, calls `handle()` method. Template specialization allows type-safe method invocation with compile-time type checking of message and channel types.

## External Dependencies
- Forward declarations: `Message`, `CommunicationsChannel` (defined elsewhere in networking subsystem)
- Standard library: `<cstdlib>` (for general heap operations, though explicit allocation only in factory)
- Conditional compilation: guarded by `#if !defined(DISABLE_NETWORKING)` from config.h
- No external dependencies beyond standard library
