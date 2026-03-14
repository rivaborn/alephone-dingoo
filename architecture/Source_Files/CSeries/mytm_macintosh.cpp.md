# Source_Files/CSeries/mytm_macintosh.cpp

## File Purpose
Implements a Macintosh-specific timer task abstraction for the Aleph One game engine, wrapping the native Timer Manager (TMTask) API. Provides periodic callback-based timers for both classic 68k Mac OS and Carbon frameworks, with platform-specific memory management handling.

## Core Responsibilities
- Wrap Macintosh Timer Manager tasks with custom callback function pointers
- Manage timer task lifecycle (setup, reset, removal)
- Handle platform-specific concerns (68k A5 register, Carbon UPP allocation)
- Execute periodic callback functions and manage re-priming
- Provide thread-safe mutex stubs for game engine integration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `myTMTask` | struct | Wraps TMTask with callback function pointer, timing state, and 68k A5 register (if needed) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `timer_upp` / `timer_desc` | `TimerUPP` / `RoutineDescriptor` | static | Universal Procedure Pointer for timer callback; platform-dependent allocation (Carbon vs classic) |

## Key Functions / Methods

### timer_proc
- **Signature:** `static pascal void timer_proc(TMTaskPtr task)`
- **Purpose:** Callback invoked by Timer Manager when timer fires; executes user callback and re-primes if needed
- **Inputs:** `task` ΓÇô pointer to myTMTask cast as TMTaskPtr
- **Outputs/Return:** void
- **Side effects:** Invokes user callback function; may call `PrimeTime` (re-arm timer) or mark `tmWakeUp` as 0 (stop); modifies `primed` flag; restores A5 on 68k
- **Calls:** User-supplied `(*mytask->func)()`, `PrimeTime`, `set_a5` / `get_a5` (68k only)
- **Notes:** On 68k, saves/restores A5 register to maintain code context. Returns immediately if callback returns false.

### myTMSetup
- **Signature:** `myTMTaskPtr myTMSetup(long time, bool (*func)(void))`
- **Purpose:** Create and register a standard timer task with the Timer Manager
- **Inputs:** `time` ΓÇô interval in machine ticks; `func` ΓÇô callback function returning bool (true to reschedule, false to stop)
- **Outputs/Return:** Pointer to allocated myTMTask, or NULL on allocation failure
- **Side effects:** Allocates memory; calls `InsTime` (register task) and `PrimeTime` (start timer); saves A5 on 68k
- **Calls:** `new`, `InsTime`, `PrimeTime`
- **Notes:** Uses InsTime (standard insertion); `insX` flag left false

### myXTMSetup
- **Signature:** `myTMTaskPtr myXTMSetup(long time, bool (*func)(void))`
- **Purpose:** Create and register an extended timer task with the Timer Manager
- **Inputs:** `time` ΓÇô interval; `func` ΓÇô callback function
- **Outputs/Return:** Pointer to allocated myTMTask, or NULL on allocation failure
- **Side effects:** Allocates memory; calls `InsXTime` (extended insertion) and `PrimeTime`; saves A5 on 68k
- **Calls:** `new`, `InsXTime`, `PrimeTime`
- **Notes:** Nearly identical to `myTMSetup` but uses `InsXTime` for extended timer support

### myTMRemove
- **Signature:** `myTMTaskPtr myTMRemove(myTMTaskPtr task)`
- **Purpose:** Remove timer from manager and deallocate task structure
- **Inputs:** `task` ΓÇô pointer to myTMTask (may be NULL)
- **Outputs/Return:** Always NULL
- **Side effects:** Calls `RmvTime` to deregister; deletes allocated task memory
- **Calls:** `RmvTime`, `delete`
- **Notes:** Safe to call with NULL (early return); return value discarded in practice

### myTMReset
- **Signature:** `void myTMReset(myTMTaskPtr task)`
- **Purpose:** Reset an existing timer task (remove from queue, re-insert, re-prime)
- **Inputs:** `task` ΓÇô pointer to myTMTask
- **Outputs/Return:** void
- **Side effects:** If primed, removes from Timer Manager queue, re-inserts via `InsTime` or `InsXTime` based on `insX` flag, and re-primes
- **Calls:** `RmvTime`, `InsTime` or `InsXTime`, `PrimeTime`
- **Notes:** No-op if task not primed; always marks task primed at end

### Dummy Functions (Stubs)
- `myTMCleanup(bool)`, `take_mytm_mutex()`, `release_mytm_mutex()`, `mytm_initialize()`
  - **Purpose:** Provide SDL/cross-platform compatibility; Classic Mac timers are atomic and require no cleanup or mutex
  - **Notes:** Mutex functions lie (`return true`) so callers don't break; marked as "WZ's dummy functions"

## Control Flow Notes
- **Initialization:** Game calls `myTMSetup` or `myXTMSetup`, which registers timer with Timer Manager
- **Runtime loop:** Timer Manager calls `timer_proc` at specified intervals; callback returns bool to indicate reschedule
- **Shutdown:** Game calls `myTMRemove` to deregister and free task
- Comment suggests Carbon should use mutex or SDL backend (not currently enforced in this Classic implementation)

## External Dependencies
- **Macintosh APIs:** `<Timer.h>` (classic) or `<Carbon/Carbon.h>` (Carbon)ΓÇöprovides `TMTask`, `InsTime`, `InsXTime`, `PrimeTime`, `RmvTime`
- **CSeries headers:** `cstypes.h` (int32, uint32), `csmisc.h` (A5 register helpers on 68k)
- **Platform conditionals:** `env68k` (68k CPU), `TARGET_API_MAC_CARBON`, `EXPLICIT_CARBON_HEADER`
- **Defined elsewhere:** User callback functions, Timer Manager runtime
