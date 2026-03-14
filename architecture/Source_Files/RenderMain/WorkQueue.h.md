# Source_Files/RenderMain/WorkQueue.h

## File Purpose
Template class implementing a FIFO work queue with object pooling via a free list. Designed for efficient reuse of elements without repeated allocation/deallocation. Originally developed for Aleph One's vis tree generator.

## Core Responsibilities
- Manage a FIFO queue of generic template type `T` using intrusive linked-list indexing
- Maintain a free pool of pre-allocated elements for reuse via chunked allocation
- Track active queue and free list separately using indices
- Provide push/pop operations with minimal memory fragmentation
- Support pre-allocation and bulk clearing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `elm` | struct | Node in the pool; holds data payload and index to next node |
| `WorkQueue<T>` | class template | FIFO queue container managing pooled elements |

## Global / File-Static State
None.

## Key Functions / Methods

### push
- Signature: `void push(const T &it)`
- Purpose: Add item to the end of the active queue
- Inputs: Const reference to item of type `T`
- Outputs/Return: None (void)
- Side effects: May allocate new elements in chunks if free list is empty; modifies internal index pointers (`eNext`, `eLast`, `eFirstFree`, `eLastFree`)
- Calls: `vector::reserve()`, `vector::push_back()`, `Debugger()`
- Notes: Caches local copies of member variables for compiler optimization. Includes invariant check: if `eLastFree == eLast`, calls `Debugger()` (sanity check that free list and active list don't overlap). Allocates in fixed chunks (`chunkSize = 4`) to amortize allocation cost.

### pop
- Signature: `T* pop()`
- Purpose: Remove and return a pointer to the front item from the active queue
- Inputs: None
- Outputs/Return: Pointer to data (`T*`), or `NULL` if queue is empty
- Side effects: Modifies internal index pointers; delinks item from active list and relinks it into free list
- Calls: `Debugger()`
- Notes: Returned pointer is guaranteed valid only until the next `push()` call. Invariant check: if `eLastFree == eLast`, calls `Debugger()`.

### flush
- Signature: `void flush()`
- Purpose: Clear all active and free list entries, reset state
- Inputs: None
- Outputs/Return: None (inline)
- Side effects: Clears underlying vector; resets all index pointers to -1

### preallocate
- Signature: `void preallocate(int size)`
- Purpose: Reserve capacity in the underlying vector to reduce future allocation cost
- Inputs: Desired size (int)
- Outputs/Return: None (inline)
- Side effects: Calls `vector::reserve()`

## Control Flow Notes
Not inferable from this file. This is a container/utility class; no explicit frame loop, render cycle, or shutdown logic is present. Integration point would be visible where `WorkQueue` instances are created and used by game systems.

## External Dependencies
- `#include <vector>`: Standard library dynamic array; used to store pool of `elm` nodes

---

**Notes on design:**
- Uses integer indices instead of pointers for compact storage and cache locality
- Two linked lists managed in single array: active queue and free list
- Compiler-specific optimization comment ("CW5 doesn't make this function suck donkeyballs") suggests performance-critical code tuned for CodeWarrior 5
- `Debugger()` appears to be a debug assertion/breakpoint macro (not defined in header)
