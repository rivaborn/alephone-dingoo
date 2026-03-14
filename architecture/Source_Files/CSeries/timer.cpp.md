# Source_Files/CSeries/timer.cpp

## File Purpose
Implements a Macintosh Classic timer system using the Time Manager toolbox. Maintains a global millisecond counter incremented by a platform timer interrupt and provides a Timer class for measuring elapsed time across application code.

## Core Responsibilities
- Manage global `globalTime` counter incremented by Mac Time Manager callbacks
- Install and initialize the Time Manager task on startup via singleton `TimerInstaller`
- Handle periodic timer interrupts and reschedule the timer task
- Provide `Timer::Clicks()` method to query elapsed time relative to a start point
- Clean up timer task and memory locks on shutdown

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Timer | class | Tracks elapsed time by storing start/stop checkpoints against `globalTime` |
| TimerInstaller | class | Singleton that owns and manages the Mac TMTask and timer callback registration |
| TMTask | struct (Mac Toolbox) | Time Manager task descriptor; fields include qLink, qType, tmAddr (callback), tmCount, tmWakeUp |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| globalTime | uint32 | global | Running millisecond counter incremented by timer callback; read by all Timer instances |
| gTimerInstaller | TimerInstaller | static (file) | Singleton instance; constructor runs on library load, destructor on unload |

## Key Functions / Methods

### Timer::Clicks()
- **Signature:** `uint32 Clicks()`
- **Purpose:** Return elapsed time in milliseconds ("clicks") since Timer was started.
- **Inputs:** None (reads member `startTime` and `totalTime`).
- **Outputs/Return:** uint32 elapsed clicks.
- **Side effects:** None.
- **Calls:** None visible in this file; reads global `globalTime`.
- **Notes:** If `startTime == 0`, Timer is stopped and returns accumulated `totalTime`. If `startTime != 0`, Timer is running; returns `totalTime + (globalTime - startTime)` to account for ongoing interval.

### TimerInstaller::TimerInstaller()
- **Signature:** Constructor (no params).
- **Purpose:** Install the Time Manager timer task on application startup.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Locks 4 bytes of memory for `globalTime`, installs TMTask via `InsXTime()`, primes task with `PrimeTime()` to fire every `MSECS_PER_CLICK` ms.
- **Calls:** `HoldMemory()`, `Gestalt()`, `NewTimerUPP()`, `InsXTime()`, `PrimeTime()`.
- **Notes:** Checks Gestalt for extended Time Manager support; silently returns if unsupported or memory lock fails. This is a global singleton; constructor runs once at module initialization.

### TimerInstaller::~TimerInstaller()
- **Signature:** Destructor (no params).
- **Purpose:** Clean up timer task and release locked memory on shutdown.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Removes TMTask via `RmvTime()`, unlocks `globalTime` memory via `UnholdMemory()`.
- **Calls:** `RmvTime()`, `UnholdMemory()`.
- **Notes:** Must be called on clean shutdown; comment in header warns that app must set terminate routine to `__terminate` to ensure proper cleanup if app crashes.

### TimerInstaller::TimerCallback() [static]
- **Signature:** `pascal void TimerCallback(TMTaskPtr tmTaskPtr)`
- **Purpose:** Interrupt handler called by Mac Time Manager at regular intervals; increments global time and reschedules itself.
- **Inputs:** `tmTaskPtr` ΓÇô pointer to the TMTask (unused in body).
- **Outputs/Return:** None.
- **Side effects:** Increments `globalTime` by 1; calls `PrimeTime()` to reschedule the next interrupt.
- **Calls:** `PrimeTime()`.
- **Notes:** Runs in interrupt context; must be fast and reentrant. `pascal` calling convention is required for Mac Toolbox callback compatibility.

## Control Flow Notes
1. **Initialization (startup):** `gTimerInstaller` singleton constructor runs, installs the TMTask and primes the timer.
2. **Per-tick (interrupt):** `TimerCallback()` fires every `MSECS_PER_CLICK` ms, increments `globalTime`, reschedules itself.
3. **Per-frame/query:** Application Timer instances call `Clicks()` to read elapsed time; the method snapshots `globalTime` against the stored `startTime`.
4. **Shutdown:** `gTimerInstaller` destructor removes the task and unlocks memory. Header warns that crash without proper `__terminate` routine will leave timer task installed and destabilize the OS.

## External Dependencies
- **Mac Toolbox headers:** `<MacMemory.h>`, `<Timer.h>`, `<Gestalt.h>`
- **Mac Toolbox functions (defined elsewhere):**
  - `HoldMemory()`, `UnholdMemory()` ΓÇô lock/unlock memory pages
  - `Gestalt()` ΓÇô query OS capabilities
  - `InsXTime()`, `RmvTime()` ΓÇô install/remove Time Manager task
  - `PrimeTime()` ΓÇô schedule next timer interrupt
  - `NewTimerUPP()` ΓÇô create universal procedure pointer for callback
