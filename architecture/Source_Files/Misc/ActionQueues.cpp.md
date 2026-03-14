# Source_Files/Misc/ActionQueues.cpp

## File Purpose
Implements a multi-player action queue manager for the Marathon: Aleph One game engine. Encapsulates circular buffers of 32-bit action flags (controller input) per player, with special handling for zombie player state controllability and queue lifecycle management.

## Core Responsibilities
- Allocate and manage contiguous memory pools for per-player action queues
- Enqueue 32-bit action flags from input/network sources into per-player circular buffers
- Dequeue and return action flags for consumption by physics/update routines
- Peek at queued actions without removal for lookahead/inspection
- Count queued actions to detect fullness and availability
- Reset individual or all player queues during level transitions or on demand
- Enforce zombie player input restrictions (controllable vs. non-controllable mode)
- Provide queue modification capability (ModifiableActionQueues subclass) for in-flight action patching

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `action_queue` | struct (nested in header) | Per-player queue metadata: read/write indices, buffer pointer |
| `ActionQueues` | class | Main container managing N players' queues with shared memory layout |
| `ModifiableActionQueues` | class (subclass of ActionQueues) | Extends ActionQueues with ability to modify flags at queue head |

## Global / File-Static State
None.

## Key Functions / Methods

### ActionQueues (constructor)
- **Signature:** `ActionQueues(unsigned int inNumPlayers, unsigned int inQueueSize, bool inZombiesControllable)`
- **Purpose:** Allocate and initialize circular queue infrastructure for a set of players.
- **Inputs:** Number of players, size of each queue (power of 2 typical), flag to enable zombie controllability.
- **Outputs/Return:** None (CTOR).
- **Side effects:** Allocates two dynamic arrays: `mQueueHeaders` (action_queue structs) and `mFlagsBuffer` (flat uint32 buffer). Initializes read/write indices to 0 for all queues. Asserts on allocation failure.
- **Calls:** `new[]` (operator).
- **Notes:** Memory layout is denseΓÇöall queues' data in one contiguous buffer, with headers pointing into segments. Modeled after original player.cpp::allocate_player_memory().

### ~ActionQueues (destructor)
- **Signature:** `~ActionQueues()`
- **Purpose:** Deallocate queue buffers and headers.
- **Outputs/Return:** None.
- **Side effects:** Frees `mFlagsBuffer` and `mQueueHeaders` if non-null.
- **Calls:** `delete[]`.

### reset()
- **Signature:** `void reset()`
- **Purpose:** Reset all player queues to empty state (read and write indices to 0).
- **Side effects:** All pending actions discarded for all players.
- **Calls:** None.
- **Notes:** Lifted from player.cpp::reset_player_queues().

### resetQueue(int inPlayerIndex)
- **Signature:** `void resetQueue(int inPlayerIndex)`
- **Purpose:** Reset a single player's queue, leaving others untouched.
- **Inputs:** Player index (0-based).
- **Side effects:** Clears actions for that player only.
- **Calls:** None.
- **Notes:** Added 2003-05-14 for LegacyActionQueueToTickBasedQueueAdapter integration. Asserts bounds.

### enqueueActionFlags(int, const uint32*, int)
- **Signature:** `void enqueueActionFlags(int player_index, const uint32 *action_flags, int count)`
- **Purpose:** Write one or more action flags into a player's queue.
- **Inputs:** Player index, array of action flags, count of flags to enqueue.
- **Outputs/Return:** None.
- **Side effects:** Advances queue's write index modulo queue size. Writes to queue buffer. Logs error if queue overflows (write catches up to read).
- **Calls:** `get_player_data()`, `PLAYER_IS_ZOMBIE()`, `logError1()`.
- **Notes:** Returns early (no-op) if target is a non-controllable zombie. Lifted from player.cpp::queue_action_flags(). Write index wraps with modulo arithmetic.

### dequeueActionFlags(int)
- **Signature:** `uint32 dequeueActionFlags(int player_index)`
- **Purpose:** Remove and return the next action flag from a player's queue.
- **Inputs:** Player index.
- **Outputs/Return:** 32-bit action flag (0 if queue empty or zombie non-controllable).
- **Side effects:** Advances read index modulo queue size.
- **Calls:** `get_player_data()`, `PLAYER_IS_ZOMBIE()`, `logError1()`.
- **Notes:** Non-controllable zombies always return 0 (special case). Logs error if queue underflow (read == write). Lifted from player.cpp::dequeue_action_flags().

### peekActionFlags(int, size_t)
- **Signature:** `uint32 peekActionFlags(int inPlayerIndex, size_t inElementsFromHead)`
- **Purpose:** Examine action flag at specified offset in queue without removal.
- **Inputs:** Player index, element offset from head (0 = next to dequeue).
- **Outputs/Return:** 32-bit action flag (0 if out of bounds or non-controllable zombie).
- **Side effects:** None (read-only).
- **Calls:** `get_player_data()`, `PLAYER_IS_ZOMBIE()`, `countActionFlags()`, `logError3()`.
- **Notes:** Added 2003-06-14. Validates offset against queue size. Logs error if peek exceeds queue depth. Reuses logic from dequeueActionFlags(); code duplication noted in comment.

### countActionFlags(int)
- **Signature:** `unsigned int countActionFlags(int player_index)`
- **Purpose:** Return number of action flags currently queued for a player.
- **Inputs:** Player index.
- **Outputs/Return:** Queue depth (0 to mQueueSize).
- **Side effects:** None.
- **Calls:** `get_player_data()`, `PLAYER_IS_ZOMBIE()`.
- **Notes:** Non-controllable zombies report full queue (mQueueSize). Uses single modulo operation for active players to avoid branching. Lifted from player.cpp::get_action_queue_size().

### zombiesControllable() / setZombiesControllable(bool)
- **Signature:** `bool zombiesControllable()` / `void setZombiesControllable(bool inZombiesControllable)`
- **Purpose:** Query or update the zombie controllability flag for this queue set.
- **Outputs/Return:** bool (zombies controllable).
- **Side effects:** setZombiesControllable() updates mZombiesControllable (affects all enqueue/dequeue/count behavior).
- **Notes:** Changed from per-call argument to per-queue-set property in 2003-02-03.

### ModifiableActionQueues::modifyActionFlags(int, uint32, uint32)
- **Signature:** `void modifyActionFlags(int inPlayerIndex, uint32 inFlags, uint32 inFlagsMask)`
- **Purpose:** Patch the action flags at the head of a player's queue (e.g., to filter or inject input).
- **Inputs:** Player index, replacement flags, bitmask indicating which bits to replace.
- **Side effects:** Modifies queue buffer at read index in-place (if queue non-empty and sentinel check passes).
- **Calls:** `countActionFlags()`, `logError()`.
- **Notes:** Subclass-only method. Skips modification if buffer is already 0xffffffff (sentinel for "do not modify"). Logs error if queue empty.

## Control Flow Notes
**Initialization:** Constructor allocates queues once per game session or per-set instantiation. Player data is externally provided via get_player_data().

**Steady-state (frame/update loop):** Input handlers call enqueueActionFlags() with new input. Physics/movement routines call dequeueActionFlags() or countActionFlags() to process queued actions. peekActionFlags() may be used by predictive update or lookahead logic.

**Zombie handling:** Non-controllable zombies are blocked at enqueue (no-op) and return 0 at dequeue, bypassing network/input processing. Controllable zombies treated as normal players (e.g., for Pfhortran scripting).

**Lifecycle:** reset() called on level transitions. setZombiesControllable() toggles zombie policy dynamically.

## External Dependencies
- **player.h**: `get_player_data()` function (returns player_data struct for a given index), `PLAYER_IS_ZOMBIE()` macro (checks zombie flag in player_data.flags).
- **Logging.h**: `logError1()`, `logError3()` macros for error reporting (queue overflow, underflow, out-of-bounds peek).
- **cseries.h** (included via ActionQueues.h): Basic types (`uint32`, `int16`, etc.).
