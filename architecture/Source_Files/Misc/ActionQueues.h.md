# Source_Files/Misc/ActionQueues.h

## File Purpose
Encapsulates a set of circular action flag queues for multiple players, enabling decoupling of input collection from input processing. Manages queueing and dequeuing of action flags (likely button/control state) with support for controllable zombie players.

## Core Responsibilities
- Manage per-player circular queues of action flags (uint32)
- Enqueue input actions from player input collection
- Dequeue actions for frame-by-frame consumption
- Peek at queued flags without removal (for debugging/lookahead)
- Track queue capacity and available space
- Control zombie player controllability as a queue-set property
- Provide modifiable queue variant for in-queue action adjustment

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ActionQueues` | class | Container managing per-player circular queues of action flags |
| `ModifiableActionQueues` | class | Subclass allowing modification of flags at queue head |
| `action_queue` | struct (protected) | Single circular queue with read/write indices and backing buffer |

## Global / File-Static State
None.

## Key Functions / Methods

### ActionQueues (constructor)
- Signature: `ActionQueues(unsigned int inNumPlayers, unsigned int inQueueSize, bool inZombiesControllable)`
- Purpose: Initialize queue set for given player count with fixed queue size
- Inputs: Player count, per-queue capacity, zombie controllability flag
- Outputs/Return: None (constructor)
- Side effects: Allocates `mQueueHeaders` array and unified `mFlagsBuffer` for all queues
- Calls: (implicit new/malloc)
- Notes: Single allocation strategy pools all flags into one buffer; read/write indices manage per-queue ranges

### enqueueActionFlags
- Signature: `void enqueueActionFlags(int inPlayerIndex, const uint32* inFlags, int inFlagsCount)`
- Purpose: Add action flag(s) to a player's queue
- Inputs: Player index, array of flags, count
- Outputs/Return: None
- Side effects: Advances write index for target queue; may overflow if queue full
- Calls: (visible in header only)
- Notes: Multiple flags can be enqueued in one call; zombie controllability may filter enqueueing

### dequeueActionFlags
- Signature: `uint32 dequeueActionFlags(int inPlayerIndex)`
- Purpose: Remove and return next action flag from queue
- Inputs: Player index
- Outputs/Return: Single uint32 action flag
- Side effects: Advances read index; returns garbage/zero if queue empty
- Calls: (visible in header only)
- Notes: One flag at a time; caller must check queue not empty with `countActionFlags()`

### peekActionFlags
- Signature: `uint32 peekActionFlags(int inPlayerIndex, size_t inElementsFromHead)`
- Purpose: Examine a queued flag without removal
- Inputs: Player index, offset from head of queue
- Outputs/Return: uint32 action flag
- Side effects: None (read-only)
- Calls: (visible in header only)
- Notes: Added June 14, 2003 for debugging/lookahead; does not advance read index

### modifyActionFlags (ModifiableActionQueues)
- Signature: `void modifyActionFlags(int inPlayerIndex, uint32 inFlags, uint32 inFlagsMask)`
- Purpose: Modify action flags at the head of queue
- Inputs: Player index, new flags value, mask indicating which bits to change
- Outputs/Return: None
- Side effects: Modifies flags in-place at head without dequeuing
- Calls: (visible in header only)
- Notes: Useful for input normalization or recording modifications before consumption

## Control Flow Notes
This module is part of the input/action flow: actions are enqueued during input polling/collection, then dequeued and processed in the main game update loop. The circular queue structure allows decoupling input rate from game frame rate. `countActionFlags()` likely serves as a ready check before `dequeueActionFlags()` in each frame's action-processing phase.

## External Dependencies
- `cseries.h` (common utility macros, types, and platform abstractions)
- Standard C types: `uint32`, `size_t`, `bool`
- Copy constructor and assignment operator explicitly disabled (private, unimplemented)
