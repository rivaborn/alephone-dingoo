# Source_Files/CSeries/csalerts.h

## File Purpose
Header file declaring alert, warning, and assertion infrastructure for the Aleph One game engine. Provides user-facing error dialogs, debug assertions, and panic/halt functions. Supports both debug and release builds with conditional macro implementations.

## Core Responsibilities
- Declare alert/dialog functions for displaying error messages to the user
- Define severity levels for categorizing alerts (info vs. fatal)
- Provide debug assertion macros that call internal assert/warn functions
- Declare halt/panic functions that terminate execution without returning
- Abstract platform-specific dialog behavior (Mac Carbon vs. SDL)
- Support both detailed (with message strings) and resource-based (with resid/item) alert variants

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| infoError, fatalError | enum constants | Severity levels for alert classification |

## Global / File-Static State
None.

## Key Functions / Methods

### alert_user (overload 1)
- Signature: `void alert_user(const char *message, short severity = infoError)`
- Purpose: Display an alert dialog with a text message and severity level
- Inputs: message (C string), severity (default infoError)
- Outputs/Return: void
- Side effects: Displays modal dialog; blocks until user dismisses
- Calls: (not visible; implemented elsewhere)
- Notes: Provides a simple message-based alert with optional severity parameter

### alert_user (overload 2)
- Signature: `void alert_user(short severity, short resid, short item, OSErr error)`
- Purpose: Display an alert using resource identifiers and OS error code
- Inputs: severity, resid (resource ID), item (dialog item index), error (OS error)
- Outputs/Return: void
- Side effects: Displays modal dialog; retrieves strings from resource fork
- Calls: (not visible; implemented elsewhere)
- Notes: Resource-based alert; integrates OS error codes

### halt
- Signature: `void halt(void) NORETURN`
- Purpose: Terminate execution immediately without returning
- Inputs: none
- Outputs/Return: never returns (marked NORETURN)
- Side effects: Abnormal process termination
- Calls: (not visible; implemented elsewhere)
- Notes: Bare panic function; typically called on unrecoverable fatal error

### vhalt
- Signature: `void vhalt(const char *message) NORETURN`
- Purpose: Terminate execution with an optional message
- Inputs: message (diagnostic string)
- Outputs/Return: never returns (marked NORETURN)
- Side effects: Displays message (likely to console/log), terminates process
- Calls: (not visible; implemented elsewhere)
- Notes: Variant of halt that logs a reason before panic

### _alephone_assert
- Signature: `void _alephone_assert(const char *file, int32 line, const char *what) NORETURN`
- Purpose: Internal handler for assertion failures in debug mode
- Inputs: file (source file path), line (line number), what (assertion condition/message)
- Outputs/Return: never returns (marked NORETURN)
- Side effects: Logs assertion failure, terminates process
- Calls: (not visible; implemented elsewhere)
- Notes: Called via assert/vassert macros; includes source location

### _alephone_warn
- Signature: `void _alephone_warn(const char *file, int32 line, const char *what)`
- Purpose: Internal handler for assertion warnings in debug mode
- Inputs: file (source file path), line (line number), what (assertion condition/message)
- Outputs/Return: void
- Side effects: Logs warning; does not terminate
- Calls: (not visible; implemented elsewhere)
- Notes: Called via warn/vwarn macros; non-fatal variant of assert

### SimpleAlert
- Signature: `DialogItemIndex SimpleAlert(AlertType Type, const char *Message, const char *Explain = NULL)` (Mac Carbon only)
- Purpose: Platform-specific simple alert dialog on Mac
- Inputs: Type (alert type), Message (main text), Explain (optional explanation)
- Outputs/Return: DialogItemIndex (which button user clicked)
- Side effects: Displays modal dialog; blocks until dismissed
- Calls: (not visible; Mac toolbox functions)
- Notes: Conditional on TARGET_API_MAC_CARBON && !SDL; allows user response

## Control Flow Notes
- **Initialization**: No startup required; header declares interfaces only.
- **Shutdown**: `halt()` / `vhalt()` are abnormal termination points.
- **Debug vs. Release**: Assertions and warnings are stripped to no-ops in non-DEBUG builds (via preprocessor macros), minimizing runtime overhead.
- **Platform Detection**: Mac Carbon dialogs vs. generic fallback; compiled conditionally.

## External Dependencies
- `OSErr` (macOS error type; defined in Mac headers)
- `DialogItemIndex`, `AlertType` (Mac Carbon API types; conditional import)
- Compiler support for `__attribute__((noreturn))` (GCC/Clang; fallback for other compilers)
- Standard C library (const char*, int32)
