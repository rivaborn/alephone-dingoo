# Source_Files/Misc/thread_priority_sdl_dummy.cpp

## File Purpose
Provides a fallback stub implementation of thread priority boosting for platforms where OS-level thread priority adjustment is not implemented. Prints a one-time warning and gracefully degrades rather than failing.

## Core Responsibilities
- Implement the `BoostThreadPriority` interface for unsupported platforms
- Print a one-time warning about missing functionality
- Return success to callers to avoid breaking downstream code

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `didPrintOutWarning` | `bool` | static | Guards the warning message so it prints only once per session |

## Key Functions / Methods

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Stub implementation that gracefully degrades on platforms lacking thread priority support.
- **Inputs:** `inThread` ΓÇô pointer to an SDL thread (not used in this dummy implementation)
- **Outputs/Return:** Always returns `true` to indicate success
- **Side effects:** Prints warning message to stdout on first call; modifies static `didPrintOutWarning` flag
- **Calls:** `printf()` (libc)
- **Notes:** Intentionally returns success despite not performing any actual priority boost, to prevent cascading errors in network code. Comment indicates network performance may suffer on platforms using this stub.

## Control Flow Notes
Called during initialization or when network features activate. The warning is printed once at runtime to alert developers/users that thread-priority optimization is unavailable on the current platform. This file is conditionally compiled only for systems without native thread priority support.

## External Dependencies
- `<stdio.h>` ΓÇô `printf()`
- `"thread_priority_sdl.h"` ΓÇô function declaration
- `SDL_Thread` (struct) ΓÇô defined elsewhere in SDL library
