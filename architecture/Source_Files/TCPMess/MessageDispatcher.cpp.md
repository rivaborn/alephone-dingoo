# Source_Files/TCPMess/MessageDispatcher.cpp

## File Purpose
Conditional compilation wrapper for the MessageDispatcher class. When `DISABLE_NETWORKING` is not defined, includes the header to enable message dispatching over TCP. The actual implementation is in the paired header file.

## Core Responsibilities
- Conditionally includes MessageDispatcher.h only when networking is enabled
- Acts as the compilation unit for the message dispatcher subsystem

## Key Types / Data Structures
None (all types defined in the paired header).

## Global / File-Static State
None.

## Key Functions / Methods
None (all methods defined in the paired header).

## Control Flow Notes
This file has no control flow logic. The actual message dispatcher operates as follows (from the header):
1. **Registration phase**: Callers register message handlers by type via `setHandlerForType()` or set a default via `setDefaultHandler()`
2. **Dispatch phase**: When `handle(Message*, CommunicationsChannel*)` is called, it looks up the handler for the message's type and invokes that handler's `handle()` method
3. **Fallback**: If no specific handler exists for a type, the default handler is used

This is a strategy pattern implementation for extensible message routing.

## External Dependencies
- **config.h** ΓÇö autoconf-generated configuration header; checks `DISABLE_NETWORKING` to conditionally enable this module
- **MessageDispatcher.h** ΓÇö contains the class definition (inherits from `MessageHandler`, uses `std::map` for routing)
- **Message.h** / **MessageHandler.h** ΓÇö defined elsewhere; provide message types and handler interface

**Notes:**  
The .cpp file is minimal; nearly all logic is in the header. This is an early-2000s codebase (2003 copyright) where template-heavy designs favored header-only patterns.
