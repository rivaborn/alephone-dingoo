# Source_Files/Misc/LocalEvents.h

## File Purpose
Defines a bitflag-based local event system for single-machine events (quit, pause, volume, UI, debug). Provides thread-safe inline functions to post and consume local events accumulated in a global flag variable.

## Core Responsibilities
- Define all local event types as hex-valued bitflags (quit, pause, sound, map zoom, inventory navigation, player/side switching, camera/view modes, debug/screenshot)
- Post local events from input devices to a global event accumulator
- Clear consumed events from the accumulator
- Atomically test and consume events (test-and-clear pattern)
- Support asynchronous input polling in a separate thread from main processing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| LocalEvent_* constants | enum constants (anonymous) | Event type identifiers as power-of-two bitflags (Quit, Pause, SoundUp/Down, MapIn/Out, InventoryNav, SwitchPlayer/Sides, ChaseCam, TunnelVision, Crosshairs, ShowPosition, Screenshot, ResetTextures, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| LocalEventFlags | uint32 | global (defined elsewhere in shell.c) | Accumulator of pending local events; each bit represents one event type |

## Key Functions / Methods

### PostLocalEvent
- Signature: `inline void PostLocalEvent(uint32 Event)`
- Purpose: Signal that a local event has occurred (set its bit in the accumulator)
- Inputs: Event (a LocalEvent_* constant, typically a power-of-two bitflag)
- Outputs/Return: void
- Side effects: Modifies global LocalEventFlags
- Calls: SET_FLAG macro
- Notes: Safe for multi-threaded use; input thread sets flags while main thread consumes them

### ClearLocalEvent
- Signature: `inline void ClearLocalEvent(uint32 Event)`
- Purpose: Manually clear/acknowledge a local event
- Inputs: Event (a LocalEvent_* constant)
- Outputs/Return: void
- Side effects: Modifies global LocalEventFlags
- Calls: SET_FLAG macro
- Notes: Rarely used directly; GetLocalEvent is the common consumption pattern

### GetLocalEvent
- Signature: `inline bool GetLocalEvent(uint32 Event)`
- Purpose: Atomically test whether an event is pending and consume it
- Inputs: Event (a LocalEvent_* constant)
- Outputs/Return: true if event was set; false otherwise
- Side effects: Clears the event flag in LocalEventFlags
- Calls: TEST_FLAG, SET_FLAG macros
- Notes: Core consumption function; used in main game loop to check for pending local events

## Control Flow Notes
Part of an input-driven event pipeline: input devices (running in a separate thread) post events via PostLocalEvent, and the main game loop repeatedly calls GetLocalEvent to consume them in a test-and-clear pattern. This avoids explicit locking by relying on atomic bitflag operations.

## External Dependencies
- `SET_FLAG`, `TEST_FLAG` macros (defined elsewhere; likely in a common header)
- `uint32` type (standard C integer type)
- `shell.c` (defines and initializes LocalEventFlags)
