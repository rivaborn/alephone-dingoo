# Source_Files/Misc/thread_priority_sdl_win32.cpp

## File Purpose
Windows-specific implementation for managing SDL thread priorities. Provides functionality to boost a thread's priority to improve responsiveness (typically for network I/O) or reduce the main thread's priority as a fallback, with version-aware fallbacks for older Windows platforms.

## Core Responsibilities
- Boost SDL thread priority to TIME_CRITICAL/HIGHEST/ABOVE_NORMAL with cascading fallbacks
- Dynamically load and invoke the Windows `OpenThread` API when available
- Reduce main thread priority to BELOW_NORMAL as a fallback strategy
- Cache main thread priority reduction to prevent redundant system calls
- Handle compatibility across Windows 98, ME, 2000, XP, and later

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OpenThreadPtrT` | typedef | Function pointer to Windows `OpenThread(DWORD, BOOL, DWORD)` for dynamic loading |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `isMainThreadPriorityReduced` | `static bool` | File-static (in `TryToReduceMainThreadPriority`) | Caches whether main thread priority has already been reduced to avoid repeated system calls |

## Key Functions / Methods

### TryToReduceMainThreadPriority
- **Signature:** `static bool TryToReduceMainThreadPriority()`
- **Purpose:** Reduce the current (main) thread's priority to BELOW_NORMAL, caching the result to prevent redundant calls.
- **Inputs:** None
- **Outputs/Return:** `true` if priority was reduced or already reduced; `false` if reduction failed.
- **Side effects:** Calls Windows API `GetCurrentThread()` and `SetThreadPriority()`; modifies static `isMainThreadPriorityReduced`.
- **Calls:** `GetCurrentThread()`, `SetThreadPriority()`
- **Notes:** Only reduces priority once, subsequent calls return cached result. Used as a fallback when boosting the target thread fails.

### BoostThreadPriority
- **Signature:** `bool BoostThreadPriority(SDL_Thread* inThread)`
- **Purpose:** Boost the specified SDL thread's priority to improve responsiveness (e.g., for network I/O), with multiple version-aware fallback strategies.
- **Inputs:** `inThread` ΓÇö pointer to SDL_Thread to boost.
- **Outputs/Return:** `true` if priority was successfully boosted; `false` if all strategies failed.
- **Side effects:** Calls Windows APIs (`GetModuleHandle`, `GetProcAddress`, `OpenThread`, `SetThreadPriority`); prints warnings to `printf` on failure.
- **Calls:** `GetModuleHandle`, `GetProcAddress`, `SDL_GetThreadID`, `OpenThread` (via function pointer), `SetThreadPriority`, `CloseHandle`, `FreeLibrary`, `TryToReduceMainThreadPriority`.
- **Notes:** 
  - **Strategy 1 (Win2000+):** Dynamically load `OpenThread` from KERNEL32, open the target thread, and set priority to TIME_CRITICAL, HIGHEST, or ABOVE_NORMAL (in cascading order).
  - **Strategy 2 (fallback, Win98+):** Cast SDL thread ID directly to HANDLE and attempt priority boost without opening the thread (undocumented but reportedly works on Win98).
  - **Strategy 3 (final fallback):** If both fail, reduce the main thread's priority instead via `TryToReduceMainThreadPriority()`.
  - Multiple `SetThreadPriority()` calls use OR logic; success occurs if *any* priority level is set.

## Control Flow Notes
This is a utility module called by other engine subsystems (likely network/I/O threading). It is not part of the main game loop and has no frame-tied behavior. Initialization/one-time setup occurs when a high-priority thread needs to be created; cleanup is minimal (just handle/library closure).

## External Dependencies
- `<windows.h>` ΓÇö Windows thread and module APIs (`GetCurrentThread`, `SetThreadPriority`, `GetModuleHandle`, `GetProcAddress`, `OpenThread`, `CloseHandle`, `FreeLibrary`, `HANDLE`, `THREAD_PRIORITY_*` constants).
- `<SDL_thread.h>` ΓÇö SDL thread type and `SDL_GetThreadID()`.
- `thread_priority_sdl.h` ΓÇö Interface declaration (extern `BoostThreadPriority`).
- `<stdio.h>` ΓÇö `printf` for warnings (defined elsewhere).
