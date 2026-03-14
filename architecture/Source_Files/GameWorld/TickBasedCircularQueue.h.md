# Source_Files/GameWorld/TickBasedCircularQueue.h

## File Purpose
Defines a family of tick-indexed circular queue data structures for storing per-frame game state (typically action flags). Supports concurrent reader/writer access patterns where the reader consumes past ticks and the writer enqueues new ticks, with careful thread-safety properties enabling lock-free usage in specific scenarios.

## Core Responsibilities
- Manage a circular buffer indexed by game tick (frame number) with separate read/write cursors
- Provide concurrent reader/writer interface with documented thread-safety invariants
- Support peeking at arbitrary ticks and consuming from the read end
- Track available capacity and queue size
- Broadcast writes to multiple child queues (DuplicatingTickBasedCircularQueue variant)
- Allow in-place mutation of enqueued elements before consumption (MutableElementsTickBasedCircularQueue variant)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| WritableTickBasedCircularQueue<tValueType> | template class (abstract interface) | Restricted write-only interface to a tick-based queue |
| DuplicatingTickBasedCircularQueue<tValueType> | template class | Decorator that broadcasts enqueue operations to multiple child queues simultaneously |
| ConcreteTickBasedCircularQueue<tValueType> | template class | Concrete implementation of tick-indexed circular buffer with separate read/write ticks |
| MutableElementsTickBasedCircularQueue<tValueType> | template class | Extends Concrete to expose mutable access to enqueued elements via `at(tick)` / `operator[]` |

## Global / File-Static State
None.

## Key Functions / Methods

### reset(int32 inTick)
- Signature: `void reset(int32 inTick)`
- Purpose: Reset both read and write tick positions to the same value (typically on initialization or queue restart)
- Inputs: `inTick` ΓÇô the tick to synchronize both pointers to
- Outputs/Return: (void)
- Side effects: Modifies mReadTick and mWriteTick; effectively discards all enqueued elements
- Calls: (none visible)
- Notes: For DuplicatingTickBasedCircularQueue, applies reset to all child queues; must not be called while reader/writer active

### enqueue(const tValueType& inFlags)
- Signature: `void enqueue(const tValueType& inFlags)`
- Purpose: Add a new element at the current write tick, advancing the write pointer
- Inputs: `inFlags` ΓÇô element value to store
- Outputs/Return: (void)
- Side effects: Modifies mWriteTick (increments by 1); writes to circular buffer at (mWriteTick % mBufferSize)
- Calls: `elementForTick(inTick)` (internal helper)
- Notes: Asserts `availableCapacity() > 0`; writer-only method; handles tick wraparound via modulo arithmetic

### dequeue()
- Signature: `void dequeue()`
- Purpose: Consume the oldest element by advancing the read tick
- Inputs: (none)
- Outputs/Return: (void)
- Side effects: Modifies mReadTick (increments by 1)
- Calls: (none visible)
- Notes: Asserts `size() > 0`; reader-only method; no data is actually freed, only the logical read position advances

### peek(int32 inTick)
- Signature: `const tValueType& peek(int32 inTick) const`
- Purpose: Return a const reference to the element at an arbitrary tick without advancing read position
- Inputs: `inTick` ΓÇô the tick to read
- Outputs/Return: Const reference to element at that tick
- Side effects: (none)
- Calls: `elementForTick(inTick)` (internal helper)
- Notes: Reader-only method; asserts `inTick >= mReadTick && inTick < mWriteTick`

### availableCapacity()
- Signature: `int32 availableCapacity() const`
- Purpose: Return the number of empty slots in the circular buffer
- Inputs: (none)
- Outputs/Return: Remaining capacity (totalCapacity - size)
- Side effects: (none)
- Calls: `totalCapacity()`, `size()` (internal queries)
- Notes: For DuplicatingTickBasedCircularQueue, returns the minimum of all child queues' capacities

### Trivial accessors
- `getReadTick()` / `getWriteTick()` ΓÇô return current read/write tick positions
- `size()` ΓÇô return `mWriteTick - mReadTick` (number of enqueued elements)
- `totalCapacity()` ΓÇô return `mBufferSize - 1` (usable capacity; buffer is sized +1 due to circular queue overhead)

## Control Flow Notes
This queue is designed for per-frame game state management in a game loop:
- **Writer** (typically input/update systems): calls `enqueue()` once per tick to add that frame's state
- **Reader** (typically simulation/network code): calls `peek()` to inspect past ticks, then `dequeue()` to advance
- The queue maintains separate cursors so the reader naturally "lags behind" the writer, consuming older frames

The file documents careful thread-safety properties: the writer can only increase size (decrease capacity), and the reader can only decrease size (increase capacity), so certain operations are safe without locks if used in the documented pattern.

## External Dependencies
- `cseries.h` ΓÇô provides type definitions (`int32`, etc.) and compatibility layer
- `<set>` ΓÇô STL container used by DuplicatingTickBasedCircularQueue to store child queue pointers
