# Source_Files/CSeries/mytm.h

## File Purpose
Header file for a lightweight task scheduler that manages timed callbacks. Supports creation, removal, and reset of scheduled tasks with millisecond-level timing, thread-safe mutex access for multi-threaded synchronization, and periodic cleanup of abandoned threads.

## Core Responsibilities
- Declare task scheduling API (`myTMSetup`, `myXTMSetup`) for registering time-based callbacks
- Provide task lifecycle management (creation, removal, reset)
- Expose thread-safe mutex primitives for synchronizing access with other subsystems
- Supply periodic cleanup mechanism for reclaiming zombie thread resources
- Offer RAII-style mutex wrapper for exception-safe locking

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `myTMTask` | Opaque struct | Represents a scheduled task; details hidden in implementation |
| `MyTMMutexTaker` | C++ class | RAII wrapper for mutex acquire/release to prevent deadlock on exception |

## Global / File-Static State
None.

## Key Functions / Methods

### myTMSetup
- Signature: `myTMTaskPtr myTMSetup(int32 time, bool (*func)(void))`
- Purpose: Create and schedule a task to execute a callback at a specified time interval
- Inputs: `time` (milliseconds), `func` (callback function returning bool)
- Outputs/Return: Pointer to allocated task (or NULL on failure)
- Side effects: Allocates task structure; may spawn/reuse thread
- Calls: Not visible in header
- Notes: Callback returns bool (likely continues if true, stops if false); extended version (`myXTMSetup`) also available

### myTMRemove
- Signature: `myTMTaskPtr myTMRemove(myTMTaskPtr task)`
- Purpose: Unschedule and deallocate a task
- Inputs: Task pointer to remove
- Outputs/Return: Task pointer (possibly NULL after removal)
- Side effects: Stops task execution; may join thread; frees memory
- Calls: Not visible in header

### myTMReset
- Signature: `void myTMReset(myTMTaskPtr task)`
- Purpose: Reset task timer to zero, deferring next execution
- Inputs: Task pointer
- Outputs/Return: None
- Side effects: Resets internal timer state

### myTMCleanup
- Signature: `void myTMCleanup(bool waitForFinishers)`
- Purpose: Reclaim resources from abandoned/zombie threads
- Inputs: `waitForFinishers` (true = block until all threads finish; false = quick pass)
- Outputs/Return: None
- Side effects: May block; deallocates orphaned thread structures
- Notes: Intended for periodic calls during main loop or shutdown

### take_mytm_mutex / release_mytm_mutex
- Signature: `bool take_mytm_mutex()` / `bool release_mytm_mutex()`
- Purpose: Acquire and release a global mutex protecting all mytm task operations
- Inputs: None
- Outputs/Return: bool (success/failure)
- Side effects: Blocks caller if lock unavailable; synchronizes with packet listening thread
- Notes: Primarily used by network layer; wrapped by `MyTMMutexTaker` for safety

### mytm_initialize
- Signature: `void mytm_initialize()`
- Purpose: Initialize the mytm subsystem (must be called before any other mytm functions)
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up internal state, mutexes, thread pool
- Notes: Called during engine startup

## Control Flow Notes
Typical usage: `mytm_initialize()` at startup ΓåÆ `myTMSetup()` / `myTMRemove()` as game events occur ΓåÆ periodic `myTMCleanup(false)` during main loop ΓåÆ `myTMCleanup(true)` at shutdown. Mutex is held by packet listening thread and game code that modifies task state concurrently.

## External Dependencies
- Standard C types: `int32`, `bool` (inferred from Aleph One codebase conventions)
- C++ `class` (RAII pattern for exception-safe locking)
- Task implementation details defined elsewhere
