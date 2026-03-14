# Source_Files/CSeries/mytm_sdl.cpp

## File Purpose
Implements an SDL-based Time Manager emulation for the Aleph One engine, providing drift-corrected periodic timers that originally used MacOS Time Manager APIs. Used primarily by networking code for recurring callbacks.

## Core Responsibilities
- Initialize and manage a global SDL mutex serializing timer task execution
- Create worker threads that execute periodic callbacks with drift correction
- Manage task lifecycle (setup, removal, reset, cleanup)
- Track outstanding tasks and reclaim zombie threads during shutdown
- Support debug-mode profiling of timing accuracy
- Coordinate thread priorities for reliable execution

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `myTMTask` | struct | Timer task state: thread, period, callback, volatile run flags, reset signal, profiling data |
| `myTMTask_profile` | struct (DEBUG only) | Timing/execution stats: call counts, drift bounds, late deadline counts, reset/resuscitation counts |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sTMTaskMutex` | `SDL_mutex*` | static | Serializes callback execution across all active timer tasks |
| `sOutstandingTasks` | `vector<myTMTaskPtr>` | static | Registry of created tasks for deferred cleanup |

## Key Functions / Methods

### mytm_initialize()
- **Signature:** `void mytm_initialize()`
- **Purpose:** One-time initialization of the global task mutex.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Creates `sTMTaskMutex` on first call; logs warning/anomaly on subsequent calls.
- **Calls:** `SDL_CreateMutex()`, logging functions.
- **Notes:** Currently relies on process exit to destroy mutex; no explicit cleanup path.

### take_mytm_mutex() / release_mytm_mutex()
- **Signature:** `bool take_mytm_mutex()`, `bool release_mytm_mutex()`
- **Purpose:** Lock/unlock the global task mutex for mutually exclusive callback execution.
- **Inputs:** None.
- **Outputs/Return:** `true` on success, `false` on lock/unlock failure.
- **Side effects:** Logs anomaly on failure. Not thread-safe (only called from high-priority worker threads or main).
- **Calls:** `SDL_LockMutex()`, `SDL_UnlockMutex()`, `SDL_GetError()`, logging.
- **Notes:** Logging from worker threads is inherently unsafe but acceptable for diagnostics.

### thread_loop() (static)
- **Signature:** `static int thread_loop(void* inData)` (entry point for SDL_Thread)
- **Purpose:** Worker thread function implementing drift-corrected periodic callback dispatch.
- **Inputs:** `inData` ΓåÆ cast to `myTMTask*`
- **Outputs/Return:** Always `0`.
- **Side effects:** Modifies task's `mResetTime`, `mKeepRunning`, `mIsRunning`, profiling data; calls user callback under mutex protection.
- **Calls:** `SDL_GetTicks()`, `SDL_Delay()`, `take_mytm_mutex()`, `release_mytm_mutex()`, user callback.
- **Notes:** 
  - Implements drift correction: `delay = period - accumulated_drift`. Negative delay signals a missed deadline.
  - Handles mid-period reset requests via busy-wait on `mResetTime > 0`.
  - Exits loop before setting `mIsRunning = false`, creating a brief race window.
  - Accepts documented risk that callback might fire once after removal due to preemption between termination check and callback invocation.

### myTMSetup() / myXTMSetup()
- **Signature:** `myTMTaskPtr myTMSetup(int32 time, bool (*func)(void))` (wrapper); `myTMTaskPtr myXTMSetup(int32 time, bool (*func)(void))` (implementation)
- **Purpose:** Create and start a periodic timer task with drift-correction.
- **Inputs:** `time` (period in milliseconds), `func` (callback returning `true` to reschedule, `false` to stop).
- **Outputs/Return:** Pointer to allocated `myTMTask` struct.
- **Side effects:** Allocates `myTMTask`, creates SDL thread, boosts thread priority, registers task in `sOutstandingTasks` vector.
- **Calls:** `new`, `SDL_CreateThread()`, `BoostThreadPriority()`, vector `push_back()`.
- **Notes:** Callback return value controls continuation; `false` terminates the task.

### myTMRemove()
- **Signature:** `myTMTaskPtr myTMRemove(myTMTaskPtr task)`
- **Purpose:** Signal a running task to stop after its next execution.
- **Inputs:** Task pointer (may be NULL).
- **Outputs/Return:** Always NULL.
- **Side effects:** Sets `mKeepRunning = false`; thread will exit naturally.
- **Calls:** None directly.
- **Notes:** Non-blocking. Documented race: callback might execute once after removal due to preemption window between termination check and callback invocation.

### myTMReset()
- **Signature:** `void myTMReset(myTMTaskPtr task)`
- **Purpose:** Reset task delay to original period or restart a dead task.
- **Inputs:** Task pointer (may be NULL).
- **Outputs/Return:** None.
- **Side effects:** If thread alive: sets `mResetTime` to signal reset. If dead: waits for thread, resets flags, creates new thread, boosts priority, increments resuscitation counter.
- **Calls:** `SDL_GetTicks()`, `SDL_WaitThread()`, `SDL_CreateThread()`, `BoostThreadPriority()`.
- **Notes:** Small race: assumes thread hasn't fully exited if `mIsRunning == true`; lazy cleanup on next reset if timing is unlucky.

### myTMCleanup()
- **Signature:** `void myTMCleanup(bool inWaitForFinishers)`
- **Purpose:** Reclaim stopped timer tasks and clean up zombie threads.
- **Inputs:** `inWaitForFinishers` ΓåÆ if `true`, wait for running tasks to finish before reaping; if `false`, skip running tasks.
- **Outputs/Return:** None.
- **Side effects:** Iterates `sOutstandingTasks`, removes stopped tasks, waits for threads, deletes task structs, dumps profiling (DEBUG).
- **Calls:** `SDL_WaitThread()`, `delete`, vector `erase()`, `myTMDumpProfile()`.
- **Notes:** Uses `erase(i--)` idiom to maintain iterator validity during loop. Called at non-time-critical moments.

### myTMDumpProfile() (DEBUG only)
- **Signature:** `void myTMDumpProfile(myTMTask* inTask)`
- **Purpose:** Log profiling statistics for a task (start/finish times, call counts, drift bounds, deadlines missed, resets, resuscitations).
- **Inputs:** Task pointer.
- **Outputs/Return:** None.
- **Side effects:** Logs via `logDump` macros.
- **Calls:** Logging macros.

## Control Flow Notes
- **Initialization:** `mytm_initialize()` creates global mutex once.
- **Per-frame/periodic:** Main thread may call `take_mytm_mutex()` / `release_mytm_mutex()` to coordinate with timer threads (e.g., for packet listening).
- **Setup:** `myTMSetup()` creates a high-priority worker thread; thread immediately enters `thread_loop()`.
- **Periodic execution:** Worker thread sleeps, recalculates drift, detects reset requests, calls user callback under mutex, exits on callback return `false` or `mKeepRunning == false`.
- **Reset:** `myTMReset()` wakes sleeping thread via `mResetTime` or restarts if dead.
- **Cleanup:** `myTMCleanup()` harvests finished tasks; typically called during shutdown or between frames.

## External Dependencies
- **SDL:** `SDL_thread.h`, `SDL_timer.h` (threads, `SDL_GetTicks()`, `SDL_Delay()`), `SDL_error.h`
- **Game engine:** `thread_priority_sdl.h` (`BoostThreadPriority()`), `mytm.h` (public interface), `cseries.h` (types, macros), `Logging.h` (logging).
- **Standard library:** `<vector>`
