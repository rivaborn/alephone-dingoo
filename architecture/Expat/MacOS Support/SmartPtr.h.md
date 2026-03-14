# Expat/MacOS Support/SmartPtr.h

## File Purpose
A template class implementing a basic scoped smart pointer for C++ that automatically deallocates the pointed-to object upon destruction. Provides pointer-like semantics (dereference, arrow operator) and explicit lifetime management methods.

## Core Responsibilities
- Wrap raw pointers and manage their lifetime
- Automatically delete pointed-to objects on destruction
- Provide operator overloads for pointer-like syntax (ΓåÆ, *, ())
- Support pointer reassignment with automatic cleanup of old pointer
- Offer explicit release/clear methods for manual ownership control
- Enable null-pointer checks via `present()`

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SmartPtr<T> | template class | Single-ownership smart pointer wrapper for type T |

## Global / File-Static State
None.

## Key Functions / Methods

### SmartPtr (constructor)
- Signature: `SmartPtr(T* _ptr = 0)`
- Purpose: Initialize smart pointer with optional raw pointer
- Inputs: Raw pointer (defaults to null)
- Outputs/Return: ΓÇö
- Side effects: Stores pointer; no deletion yet
- Calls: ΓÇö
- Notes: Null pointer is valid initial state

### ~SmartPtr (destructor)
- Signature: `~SmartPtr()`
- Purpose: Clean up by deleting pointed-to object
- Inputs: ΓÇö
- Outputs/Return: ΓÇö
- Side effects: Deletes `ptr` if non-null
- Calls: `clear_ptr()`
- Notes: Critical for lifetime management; called on scope exit

### operator= (assignment)
- Signature: `T* operator=(T* _ptr)`
- Purpose: Reassign pointer, deleting old object
- Inputs: New raw pointer
- Outputs/Return: New pointer value
- Side effects: Deletes old `ptr`, stores new one
- Calls: `clear_ptr()`
- Notes: Supports chaining (e.g., `p1 = p2 = nullptr`)

### operator-> / operator* / operator() (access)
- Signature: `T* operator->()`, `T& operator*()`, `T* operator()()`
- Purpose: Dereference pointer for member/value access
- Inputs: ΓÇö
- Outputs/Return: Pointer or reference to object
- Side effects: None
- Calls: ΓÇö
- Notes: All return raw accessors; no bounds checking

### operator== / operator!= (comparison)
- Signature: `bool operator==(T* _ptr)`, `bool operator!=(T* _ptr)`
- Purpose: Compare stored pointer with another
- Inputs: Raw pointer to compare
- Outputs/Return: Boolean equality/inequality result
- Side effects: None
- Calls: ΓÇö

### present / accept / release / clear (utility methods)
- **present()**: Null-check; returns `true` if pointer is non-null
- **accept(T* _ptr)**: Reassign and return pointer (verbose form of `operator=`)
- **release()**: Return pointer and nullify internal `ptr` (transfer ownership)
- **clear()**: Delete and nullify pointer
- Calls: `clear_ptr()`

## Control Flow Notes
Utility class, not directly part of game loop. Used throughout engine for automatic memory management. Typical usage: wrap heap-allocated objects at creation, let scope exit trigger deletion.

## External Dependencies
- Standard C++: `new`, `delete` operators
- Expat library context: inferred from path; no explicit dependencies visible in this file
