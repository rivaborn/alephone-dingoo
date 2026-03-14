# Source_Files/Misc/game_errors.cpp

## File Purpose
Simple singleton error state management system for the game engine. Maintains the last error type and code encountered during gameplay, with functions to query, set, and clear the error state.

## Core Responsibilities
- Store and update the current error type and code as they occur
- Provide read-only access to the current error state and type
- Check whether an error is pending (non-zero)
- Clear error state for recovery or state reset

## Key Types / Data Structures
From `game_errors.h`:

| Name | Kind | Purpose |
|------|------|---------|
| `systemError`, `gameError` | enum | Error type categories |
| `errNone`, `errMapFileNotSet`, `errIndexOutOfRange`, etc. | enum | Specific game error codes |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `last_type` | short | static | Currently active error type (systemError or gameError) |
| `last_error` | short | static | Error code of the most recent error (0 = none) |

## Key Functions / Methods

### set_game_error
- **Signature:** `void set_game_error(short type, short error_code)`
- **Purpose:** Update the global error state with a new error type and code.
- **Inputs:** `type` (error category), `error_code` (specific error)
- **Outputs/Return:** None
- **Side effects:** Modifies `last_type` and `last_error` statics
- **Calls:** None (other than assert in DEBUG mode)
- **Notes:** Validates `type` is in range [0, NUMBER_OF_TYPES); validates `error_code` in DEBUG if type is gameError

### get_game_error
- **Signature:** `short get_game_error(short *type)`
- **Purpose:** Retrieve the current error code and optionally the error type.
- **Inputs:** `type` (optional out-param; NULL-safe)
- **Outputs/Return:** Returns `last_error` code; writes error type via `type` pointer if provided
- **Side effects:** None
- **Calls:** None

### error_pending
- **Signature:** `bool error_pending(void)`
- **Purpose:** Check if a non-zero error is currently recorded.
- **Inputs:** None
- **Outputs/Return:** Returns true if `last_error != 0`, false otherwise
- **Side effects:** None
- **Calls:** None

### clear_game_error
- **Signature:** `void clear_game_error(void)`
- **Purpose:** Reset error state to no error.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `last_error` and `last_type` to 0
- **Calls:** None

## Control Flow Notes
Typical usage: errors are set when conditions fail during game logic, checked with `error_pending()` or `get_game_error()` by higher-level systems, then cleared via `clear_game_error()` after handling. Acts as a simple event-less error register.

## External Dependencies
- `cseries.h` ΓÇö standard cross-platform definitions (assert, types)
- `game_errors.h` ΓÇö enum definitions for error types and codes
