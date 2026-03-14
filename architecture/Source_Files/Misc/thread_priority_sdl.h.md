# Source_Files/Misc/thread_priority_sdl.h

## File Purpose
Declares the interface for boosting thread priority in the Aleph One game engine using SDL. Allows the main thread to elevate worker thread priority or, as a fallback, reduce its own priority to allow other threads to execute.

## Core Responsibilities
- Declare the `BoostThreadPriority` function for cross-platform thread priority management
- Provide abstraction over platform-specific thread scheduling APIs via SDL

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SDL_Thread | struct (forward decl.) | SDL's opaque thread handle |

## Global / File-Static State
None.

## Key Functions / Methods

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread);`
- **Purpose:** Elevate the priority of a worker thread, or fall back to reducing the main thread's priority if priority boosting is unavailable.
- **Inputs:** `inThread` ΓÇô pointer to SDL_Thread to boost
- **Outputs/Return:** `bool` ΓÇô success indicator (true if priority adjusted, false otherwise)
- **Side effects:** Modifies OS-level thread scheduling priority; affects process resource contention.
- **Calls:** Implementation details not visible in header.
- **Notes:**
  - Must be called by the main thread to allow main-thread priority reduction.
  - Repeated calls should not further reduce main thread priority (caller responsible for tracking).
  - Implements a fallback strategy: boost worker ΓåÆ reduce main (if boost fails).

## Control Flow Notes
Typical usage: called during initialization or when spawning background worker threads to ensure they get adequate CPU time relative to the main game loop thread.

## External Dependencies
- **SDL:** Forward declares `SDL_Thread` (defined in SDL headers, linked at runtime).
