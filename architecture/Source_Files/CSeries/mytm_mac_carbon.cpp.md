# Source_Files/CSeries/mytm_mac_carbon.cpp

## File Purpose
Implements a repeating timer system for macOS using Carbon Events API. Manages timer callbacks that execute at specified intervals, allowing callbacks to control timer lifecycle by returning true (continue) or false (stop).

## Core Responsibilities
- Create and install repeating timers via macOS Carbon Event Loop
- Execute user-provided callback functions at millisecond intervals
- Track timer state (primed/active vs. inactive)
- Remove timers and deallocate resources
- Support reinstalling inactive timers via reset operation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `myTMTask` | struct | Holds timer reference, callback function pointer, interval time (ms), and primed state |
| `myTMTaskPtr` | typedef | Pointer to `myTMTask` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `timer_upp` | `EventLoopTimerUPP` | static | Universal procedure pointer for the timer callback; created once and reused for all timer instances |

## Key Functions / Methods

### timer_proc
- **Signature:** `static pascal void timer_proc(EventLoopTimerRef inTimer, void *task)`
- **Purpose:** Carbon Event Loop timer callbackΓÇöinvoked when timer fires.
- **Inputs:** `inTimer` (the firing timer reference), `task` (opaque pointer to `myTMTask` instance)
- **Outputs/Return:** None (void).
- **Side effects:** Sets `primed = false` immediately; calls user callback; if callback returns false, removes the timer from event loop; if true, leaves timer installed.
- **Calls:** User-provided `mytask->func()`, `RemoveEventLoopTimer()`.
- **Notes:** Callback return value controls timer lifecycle: `true` = keep timer running, `false` = remove timer and stop iteration.

### myTMSetup
- **Signature:** `myTMTaskPtr myTMSetup(int32 time, bool (*func)(void))`
- **Purpose:** Creates and installs a new repeating timer.
- **Inputs:** `time` (interval in milliseconds), `func` (callback function returning bool).
- **Outputs/Return:** Pointer to new `myTMTask` instance (or NULL on allocation failure).
- **Side effects:** Allocates `myTMTask` on heap; installs timer in main event loop with interval `time`; sets `primed = true`.
- **Calls:** `InstallEventLoopTimer()`.
- **Notes:** Timer fires immediately at `time` ms and repeats at `time` ms intervals.

### myXTMSetup
- **Signature:** `myTMTaskPtr myXTMSetup(int32 time, bool (*func)(void))`
- **Purpose:** Wrapper for `myTMSetup`; likely for API compatibility or alternate naming convention.
- **Inputs:** Same as `myTMSetup`.
- **Outputs/Return:** Result of `myTMSetup()`.

### myTMRemove
- **Signature:** `myTMTaskPtr myTMRemove(myTMTaskPtr task)`
- **Purpose:** Removes and deallocates a timer task.
- **Inputs:** `task` (timer to remove, may be NULL).
- **Outputs/Return:** Always returns NULL.
- **Side effects:** If task is primed, removes from event loop; deallocates `task` via `delete`.
- **Calls:** `RemoveEventLoopTimer()`.
- **Notes:** Safe to call with NULL pointer.

### myTMReset
- **Signature:** `void myTMReset(myTMTaskPtr task)`
- **Purpose:** Reactivates an inactive timer or adjusts fire time of active timer.
- **Inputs:** `task` (timer to reset).
- **Outputs/Return:** None (void).
- **Side effects:** If primed, adjusts next fire time via `SetEventLoopTimerNextFireTime()`; if not primed, reinstalls timer and sets `primed = true`.
- **Calls:** `SetEventLoopTimerNextFireTime()` or `InstallEventLoopTimer()`.
- **Notes:** Handles two cases: timer already running (reschedule) vs. timer was stopped by callback (reinject into event loop).

### myTMCleanup
- **Signature:** `void myTMCleanup(bool)`
- **Purpose:** Platform-specific cleanup hook (declared in mytm.h but unimplemented for Carbon).
- **Inputs:** Boolean flag (unused on this platform).
- **Outputs/Return:** None (void).
- **Side effects:** None.
- **Notes:** Marked "WZ's dummy function"; real cleanup implemented on other platforms (e.g., SDL).

## Control Flow Notes
**Initialization:** `myTMSetup()` allocates task and installs timer in main event loop.  
**Runtime:** Carbon Event Loop invokes `timer_proc()` at intervals; callback determines continuance.  
**Removal:** `myTMRemove()` deinstalls and frees task.  
**Reset/Reactivation:** `myTMReset()` reinjects stopped timers or reschedules active ones.

## External Dependencies
- **Carbon/Carbon.h:** Event Loop API (`GetMainEventLoop`, `InstallEventLoopTimer`, `RemoveEventLoopTimer`, `SetEventLoopTimerNextFireTime`, `EventLoopTimerRef`, `EventLoopTimerUPP`, `NewEventLoopTimerUPP`, `kEventDurationMillisecond`, `kDurationMillisecond`).
- **cstypes.h:** Type definitions (`int32`, `bool`).
- **csmisc.h:** Miscellaneous utilities (unused in this file).
- **mytm.h:** Function declarations exported to rest of codebase.
