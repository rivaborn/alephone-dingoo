# Source_Files/CSeries/timer.h

## File Purpose
Provides a `Timer` class for measuring elapsed time intervals in a game engine. Tracks time using a global millisecond counter and allows accumulating multiple start/stop cycles. Includes a platform-specific warning about Mac crash handling.

## Core Responsibilities
- Declare a global monotonic time counter (`globalTime`) updated by the engine
- Provide a `Timer` class to measure elapsed time across one or more intervals
- Convert elapsed time from clicks (milliseconds) to seconds
- Accumulate total time from multiple Start/Stop cycles

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Timer | class | Interval timer with start/stop semantics and accumulated time tracking |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| globalTime | uint32 (extern) | global | Engine's monotonic time counter, incremented every millisecond |
| MSECS_PER_CLICK | macro (= 1) | file | Conversion factor; 1 click = 1 millisecond |

## Key Functions / Methods

### Timer (constructor)
- **Signature:** `Timer()`
- **Purpose:** Initialize a new timer
- **Side effects:** Calls `Clear()` to zero both `totalTime` and `startTime`
- **Notes:** Inline constructor; creates a stopped, zero-elapsed timer

### Clear()
- **Signature:** `void Clear()`
- **Purpose:** Reset accumulated and current start times to zero
- **Side effects:** Resets `totalTime` and `startTime` to 0

### Start()
- **Signature:** `void Start()`
- **Purpose:** Begin timing a new interval
- **Side effects:** Captures current `globalTime` in `startTime`

### Stop()
- **Signature:** `void Stop()`
- **Purpose:** End the current interval and accumulate elapsed time
- **Side effects:** Adds `(globalTime - startTime)` to `totalTime`; clears `startTime` to 0
- **Notes:** Safe to call multiple times (subsequent calls add 0 if `startTime == 0`)

### Clicks()
- **Signature:** `uint32 Clicks()`
- **Purpose:** Return accumulated time in milliseconds (clicks)
- **Outputs/Return:** Total elapsed time since creation or last Clear
- **Notes:** Implementation not visible; defined elsewhere (inline or .cpp file)

### Seconds()
- **Signature:** `float Seconds()`
- **Purpose:** Convert accumulated time to floating-point seconds
- **Outputs/Return:** `Clicks() / (1000.0 / MSECS_PER_CLICK)` Γëê `Clicks() / 1000.0`
- **Notes:** Inline; assumes MSECS_PER_CLICK = 1

## Control Flow Notes
Typical usage: engine updates `globalTime` each frame; game code calls `Start()` before a timed operation, `Stop()` after, then queries `Seconds()` or `Clicks()`. Multiple Start/Stop pairs accumulate, enabling measurement across non-contiguous intervals.

## External Dependencies
- `#include <cstdint>` or equivalent (implicit; `uint32` type)
- `globalTime` declared elsewhere (engine core)
- `Clicks()` implemented elsewhere

## Notes
- **Platform warning:** Macintosh-specific; app must set crash handler to `__terminate` for stability (likely pre-OS X code).
- **Missing inline:** `Clicks()` is declared but not defined; implementation is external.
- **No thread safety:** Not designed for concurrent access; assumes single-threaded game loop.
