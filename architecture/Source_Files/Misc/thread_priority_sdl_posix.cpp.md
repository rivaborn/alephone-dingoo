# Source_Files/Misc/thread_priority_sdl_posix.cpp

## File Purpose
POSIX-specific implementation for elevating SDL thread priorities. Abstracts platform-specific pthread scheduling calls to maximize thread priority for performance-critical game threads (typically audio or network I/O).

## Core Responsibilities
- Convert SDL thread handles to native POSIX pthread identifiers
- Query current scheduling policy and parameters for a target thread
- Maximize thread priority within its policy constraints
- Return graceful failure if priority boost is unsupported or denied by the system

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SDL_Thread` | struct (external) | SDL abstraction for platform threads |
| `pthread_t` | typedef (external) | POSIX thread identifier |
| `struct sched_param` | struct (external) | POSIX scheduling parameters (e.g., `sched_priority`) |

## Global / File-Static State
None.

## Key Functions / Methods

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Attempt to maximize the scheduling priority of an SDL thread on POSIX systems.
- **Inputs:** `inThread` ΓÇô SDL thread handle to boost
- **Outputs/Return:** `bool` ΓÇô `true` if successful or if priority scheduling is unavailable; `false` if pthread calls fail
- **Side effects:** 
  - Modifies kernel-level thread scheduling priority
  - May block briefly on scheduler locks
- **Calls (direct visible in this file):**
  - `SDL_GetThreadID()` ΓÇô extract pthread_t from SDL_Thread wrapper
  - `pthread_getschedparam()` ΓÇô retrieve current policy and priority
  - `sched_get_priority_max()` ΓÇô query maximum priority for the policy
  - `pthread_setschedparam()` ΓÇô apply new priority to thread
- **Notes:**
  - Only executes if `_POSIX_PRIORITY_SCHEDULING` is defined; otherwise returns `true` unconditionally (silent no-op for non-POSIX platforms)
  - Preserves the thread's current scheduling policy; only raises its priority to maximum
  - Returns `false` only if `pthread_getschedparam()` or `pthread_setschedparam()` explicitly fail (e.g., insufficient permissions)
  - Per header comment, intended for audio/I/O threads; main thread priority reduction happens elsewhere

## Control Flow Notes
Likely called during engine initialization to prioritize critical background threads (audio processing, network). No frame-level or per-update calls inferred. Conditional compilation (`#if _POSIX_PRIORITY_SCHEDULING`) provides graceful degradation on systems without priority scheduling support.

## External Dependencies
- **Includes:**
  - `<SDL/SDL_thread.h>` (or `<SDL/SDL_Thread.h>` on Mac Carbon/Mach)
  - `<pthread.h>` ΓÇô POSIX threading API
  - `<sched.h>` ΓÇô POSIX scheduling policy and priority constants
  - `"thread_priority_sdl.h"` ΓÇô interface contract
- **External symbols used:** `SDL_GetThreadID`, `pthread_getschedparam`, `sched_get_priority_max`, `pthread_setschedparam` (all from standard libraries)
