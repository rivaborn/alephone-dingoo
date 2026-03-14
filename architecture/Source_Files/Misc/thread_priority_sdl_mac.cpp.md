# Source_Files/Misc/thread_priority_sdl_mac.cpp

## File Purpose
Provides a macOS-specific stub implementation for thread priority boosting in the Aleph One game engine. The function is a no-op on Mac Classic, always returning success to maintain compatibility with the cross-platform thread priority interface.

## Core Responsibilities
- Implement `BoostThreadPriority()` for macOS platform
- Satisfy the interface contract defined in `thread_priority_sdl.h`
- Return success status without performing actual priority adjustment

## Key Types / Data Structures
None (uses SDL_Thread defined elsewhere).

## Global / File-Static State
None.

## Key Functions / Methods

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Attempt to increase a worker thread's scheduling priority on macOS, or reduce main thread priority if boosting is unavailable.
- **Inputs:** `inThread` ΓÇö SDL thread handle to boost
- **Outputs/Return:** Always `true` (success)
- **Side effects:** None; stub implementation
- **Calls:** None
- **Notes:** Currently a no-op. Commented-out code shows a warning was originally planned. Header documentation states this should be called by the main thread to allow priority reduction if boost is unavailable.

## Control Flow Notes
Called from elsewhere in the engine (likely network thread setup) when worker threads require performance priority. On macOS, this call succeeds silently without changing thread priorities, potentially impacting network performance as the header comment warns.

## External Dependencies
- `<stdio.h>` ΓÇö included but unused in current implementation
- `thread_priority_sdl.h` ΓÇö defines `BoostThreadPriority` interface
- `SDL_Thread` ΓÇö SDL library thread type (defined elsewhere)
