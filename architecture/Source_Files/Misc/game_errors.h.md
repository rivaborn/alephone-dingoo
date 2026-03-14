# Source_Files/Misc/game_errors.h

## File Purpose
Defines error types and error codes for the game engine, along with declarations for a global error-handling API. Supports both system-level and game-specific error classification.

## Core Responsibilities
- Enumerate error types (system vs. game errors)
- Define game-specific error codes (file not found, out of range, version mismatch, etc.)
- Declare the error state management API (set, get, check, clear)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| systemError, gameError | enum constant | Classify error originΓÇösystem-level or game-level |
| errNone, errMapFileNotSet, errIndexOutOfRange, errTooManyOpenFiles, errUnknownWadVersion, errWadIndexOutOfRange, errServerDied, errUnsyncOnLevelChange | enum constant | Specific game error conditions (map loading, file limits, WAD format, sync, etc.) |

## Global / File-Static State
None.

## Key Functions / Methods

### set_game_error
- Signature: `void set_game_error(short type, short error_code);`
- Purpose: Register an error with classification type and specific error code
- Inputs: `type` (systemError or gameError), `error_code` (e.g., errMapFileNotSet)
- Outputs/Return: None
- Side effects: Updates global error state
- Calls: Not inferable from this file
- Notes: Caller should use enums defined in this header; both parameters are shorts

### get_game_error
- Signature: `short get_game_error(short *type);`
- Purpose: Retrieve the current error code and write its type to caller's buffer
- Inputs: `type` (pointer to short for output)
- Outputs/Return: Error code (short); type written via output parameter
- Side effects: Reads global error state
- Calls: Not inferable from this file
- Notes: Type returned by pointer, not by return value

### error_pending
- Signature: `bool error_pending(void);`
- Purpose: Query whether an error is currently set
- Inputs: None
- Outputs/Return: Boolean (true = error set, false = no error)
- Side effects: Reads global error state
- Calls: Not inferable from this file

### clear_game_error
- Signature: `void clear_game_error(void);`
- Purpose: Clear the current error state
- Inputs: None
- Outputs/Return: None
- Side effects: Resets global error state
- Calls: Not inferable from this file

## Control Flow Notes
Not inferable from header. Likely used to store and propagate errors during initialization, resource loading, or networkingΓÇöchecked by calling code at logical decision points.

## External Dependencies
- Implicit: C standard library (bool type)
- No explicit includes
