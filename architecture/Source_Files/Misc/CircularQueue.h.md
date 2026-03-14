# Source_Files/Misc/CircularQueue.h

## File Purpose
Template-based circular queue (ring buffer) implementation for the Aleph One engine. Provides fixed-capacity FIFO storage with wraparound indices, supporting copy/assign semantics for safe queue duplication.

## Core Responsibilities
- Manage fixed-size circular buffer with dynamic (re)allocation
- Track read/write indices and handle modulo wraparound
- Provide enqueue/dequeue operations with overflow/underflow guards
- Support copy construction and assignment operators
- Calculate occupancy and remaining capacity
- Peek at front element without removal

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CircularQueue\<T\> | Template class | Generic FIFO queue storing elements of type T in a circular buffer |

## Global / File-Static State
None.

## Key Functions / Methods

### CircularQueue() (default constructor)
- Signature: `CircularQueue() : mData(NULL) { reset(0); }`
- Purpose: Initialize empty queue with no allocated storage
- Inputs: None
- Outputs/Return: Initialized object
- Side effects: Sets all indices to 0, mData to NULL

### CircularQueue(unsigned int inSize) (sized constructor)
- Signature: `explicit CircularQueue(unsigned int inSize)`
- Purpose: Initialize queue with pre-allocated capacity
- Inputs: `inSize` ΓÇô desired element capacity
- Outputs/Return: Initialized object with allocated buffer
- Side effects: Allocates `inSize + 1` elements via `new[]`

### operator= (assignment)
- Signature: `CircularQueue<T>& operator=(const CircularQueue<T>& o)`
- Purpose: Copy queue contents and capacity from another queue
- Inputs: `o` ΓÇô source queue (must be stable during copy)
- Outputs/Return: Reference to this
- Side effects: Deallocates old buffer, allocates new buffer, copies all enqueued elements
- Calls: `reset()`, `getCountOfElements()`, `enqueue()`, `getReadIndex()`
- Notes: Iterates source queue and re-enqueues each element to maintain order

### reset(unsigned int inSize)
- Signature: `void reset(unsigned int inSize)`
- Purpose: Clear queue and reallocate buffer if needed
- Inputs: `inSize` ΓÇô new capacity (0 allowed)
- Outputs/Return: None
- Side effects: Deallocates existing buffer (if any), allocates new buffer of size `inSize + 1`, resets indices to 0
- Notes: Stores `inSize + 1` to allow distinguishing full from empty state; guards against wraparound with assert

### enqueue(const T& inData)
- Signature: `void enqueue(const T& inData)`
- Purpose: Add element to queue tail
- Inputs: `inData` ΓÇô element to add
- Outputs/Return: None
- Side effects: Writes to buffer at write index, advances write index
- Calls: `getWriteIndex()`, `advanceWriteIndex()`
- Notes: Asserts that remaining space > 0 (caller must check)

### dequeue(unsigned int inAmount = 1)
- Signature: `void dequeue(unsigned int inAmount = 1)`
- Purpose: Remove one or more elements from queue head
- Inputs: `inAmount` ΓÇô number of elements to remove (default 1)
- Outputs/Return: None
- Side effects: Advances read index by amount
- Calls: `advanceReadIndex()`
- Notes: Asserts that count >= inAmount

### peek() const
- Signature: `const T& peek() const`
- Purpose: Return reference to front element without removal
- Inputs: None
- Outputs/Return: Const reference to element at read index
- Calls: `getReadIndex()`
- Notes: Caller must check `getCountOfElements() > 0` first

### Notes
- **getCountOfElements()**, **getRemainingSpace()**, **getTotalSpace()**: Simple capacity queries using modulo arithmetic
- **getReadIndex()**, **getWriteIndex()** (protected): Compute actual buffer indices with optional offset and wraparound; bounds-checked with asserts
- **advanceReadIndex()**, **advanceWriteIndex()** (protected): Move indices forward with modulo wrap and capacity validation

## Control Flow Notes
This is a utility data structure with no inherent lifecycle. Client code drives all operations (enqueue ΓåÆ dequeue cycles). Used during engine runtime wherever bounded FIFO buffering is needed (e.g., network queues, input buffers).

## External Dependencies
- `<cassert>` (for runtime assertions on bounds)
- `new[]` / `delete[]` (C++ memory allocation)
- No engine-specific includes visible; purely self-contained template
