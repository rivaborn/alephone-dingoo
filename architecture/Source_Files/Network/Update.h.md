# Source_Files/Network/Update.h

## File Purpose
Declares the `Update` singleton class for asynchronous online version checking in Aleph One. Manages update availability status and provides access to new version information when available.

## Core Responsibilities
- Implements singleton pattern for centralized update management
- Tracks update check status (checking, failed, available, unavailable)
- Runs version check asynchronously in a separate SDL thread
- Stores and exposes new version information
- Guards all functionality behind networking compile flag

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Status` | enum | Represents update check state: `CheckingForUpdate`, `UpdateCheckFailed`, `UpdateAvailable`, `NoUpdateAvailable` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `Update*` | static | Singleton instance pointer |
| `m_status` | `Status` | member | Current update check status |
| `m_new_date_version` | `std::string` | member | Version timestamp (name suggests possible typo) |
| `m_new_display_version` | `std::string` | member | Human-readable version string for UI display |
| `m_thread` | `SDL_Thread*` | member | Thread handle for async update checking |

## Key Functions / Methods

### instance()
- Signature: `static Update* instance()`
- Purpose: Returns singleton instance, creating it if necessary
- Inputs: None
- Outputs/Return: Pointer to `Update` singleton
- Side effects: Creates singleton on first call (heap allocation)
- Calls: Implicit `new Update()` on first call
- Notes: Lazy initialization; not thread-safe

### GetStatus()
- Signature: `Status GetStatus()`
- Purpose: Queries current update check state
- Inputs: None
- Outputs/Return: Current `Status` enum value
- Side effects: None
- Calls: None
- Notes: Non-blocking read of current state

### NewDisplayVersion()
- Signature: `std::string NewDisplayVersion()`
- Purpose: Returns human-readable version string for UI
- Inputs: None
- Outputs/Return: `m_new_display_version` string
- Side effects: None
- Calls: `assert(m_status == UpdateAvailable)`
- Notes: Precondition: only valid when status is `UpdateAvailable`; will assert otherwise

### Private methods
- `StartUpdateCheck()`: Initiates the version check (likely called from constructor or public API)
- `update_thread(void*)`: Static thread entry point; routes to instance `Thread()` method
- `Thread()`: Instance method executed in spawned thread; performs actual HTTP check

## Control Flow Notes
Fits into a **background polling/initialization** pattern: Constructor likely calls `StartUpdateCheck()`, which spawns `update_thread()` as a detached SDL thread. Main thread can poll `GetStatus()` and `NewDisplayVersion()` asynchronously. Designed to not block game startup or main loop.

## External Dependencies
- `#include <string>` ΓÇö `std::string` for version storage
- `#include <SDL_thread.h>` ΓÇö `SDL_Thread` for async execution
- `#include "cseries.h"` ΓÇö Aleph One utilities (likely assertion macros)
- `config.h` ΓÇö Conditional `DISABLE_NETWORKING` guard
- **Defined elsewhere**: Implementation of `StartUpdateCheck()`, `Thread()`, and actual HTTP networking (likely in `.cpp` counterpart)
