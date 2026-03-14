# Source_Files/Misc/thread_priority_sdl_macosx.cpp

## File Purpose
Implements macOS-specific thread priority boosting for SDL threads using POSIX pthread and scheduling APIs. Provides the runtime mechanism to elevate a game thread to maximum priority within its scheduling policy.

## Core Responsibilities
- Extract POSIX thread identifier from SDL_Thread wrapper
- Query current scheduling policy and parameters of the target thread
- Elevate thread priority to the maximum allowed by its scheduling policy
- Apply updated scheduling parameters back to the thread
- Handle POSIX API errors gracefully and return success/failure status

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SDL_Thread` | struct (opaque) | SDL cross-platform thread wrapper |
| `pthread_t` | typedef | POSIX thread identifier |
| `struct sched_param` | struct | POSIX scheduling parameters (contains `sched_priority` field) |

## Global / File-Static State
None

## Key Functions

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Maximize the scheduling priority of the specified SDL thread on macOS
- **Inputs:** `inThread` ΓÇô pointer to SDL_Thread to boost
- **Outputs/Return:** `bool` ΓÇô `true` on success; `false` if any pthread operation fails
- **Side effects:** Modifies kernel-level thread scheduling priority; requires appropriate process permissions
- **Calls:**
  - `SDL_GetThreadID(inThread)` ΓÇô unwrap SDL_Thread to pthread_t
  - `pthread_getschedparam(theTargetThread, &theSchedulingPolicy, &theSchedulingParameters)` ΓÇô retrieve current policy and params
  - `sched_get_priority_max(theSchedulingPolicy)` ΓÇô fetch maximum priority for the policy
  - `pthread_setschedparam(theTargetThread, theSchedulingPolicy, &theSchedulingParameters)` ΓÇô apply new params
- **Notes:**
  - Tested on macOS 10.1.0 (comment-annotated)
  - Preserves existing scheduling policy; only maximizes priority
  - Early-exit error handling: returns `false` immediately if `pthread_getschedparam` or `pthread_setschedparam` fails
  - Platform-specific; macOS/Darwin only

## Control Flow Notes
Not part of the main frame loop. Called during engine initialization or dynamic workload rebalancing to prioritize game-critical threads (e.g., physics, AI, audio). Typically invoked once per important thread at startup. Complements the header comment's note that this should be called by the main thread (priority reduction of main thread is handled separately, not in this file).

## External Dependencies
- `<SDL_Thread.h>` ΓÇô SDL threading abstraction (defines `SDL_Thread`, `SDL_GetThreadID`)
- `<pthread.h>` ΓÇô POSIX threading (`pthread_t`, `pthread_getschedparam`, `pthread_setschedparam`)
- `<sched.h>` ΓÇô POSIX scheduling (`sched_param`, `sched_get_priority_max`)
- `"thread_priority_sdl.h"` ΓÇô Local header with function declaration
- Presumed companion files for Windows and Linux/generic SDL platforms exist
