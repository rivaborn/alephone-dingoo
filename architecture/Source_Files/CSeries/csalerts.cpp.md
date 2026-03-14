# Source_Files/CSeries/csalerts.cpp

## File Purpose
Implements user-facing alert dialogs and error/assertion handling for the Aleph One game engine. Bridges platform-specific alert APIs (Carbon/Classic Mac) and logs messages to both UI and logging systems during runtime errors, warnings, and fatal shutdowns.

## Core Responsibilities
- Display modal alert dialogs with severity levels (info, fatal)
- Log errors, warnings, and assertions to both UI and logging system
- Handle assertion failures and debug warnings with file/line context
- Gracefully shut down the engine on fatal errors (cleanup + ExitToShell)
- Support dual platform implementations (Carbon vs. Classic Mac OS)
- Format error/assertion messages with csprintf and manage message buffers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| AlertType | enum (from Carbon.h) | Alert visual style (kAlertNoteAlert, kAlertStopAlert) |
| DialogItemIndex | typedef (from Carbon.h) | Button index returned by StandardAlert |
| Str255 | typedef (from Macintosh) | Pascal string: 1 length byte + 255 chars max |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| alert_text | Str255 | static | Message buffer for Classic Mac alerts (not used in Carbon) |
| assert_text | char[256] | static | Message buffer for formatting assertion/warning messages |

## Key Functions / Methods

### alert_user (char*, short)
- Signature: `void alert_user(char *message, short severity)`
- Purpose: Display a simple alert dialog with the given severity level.
- Inputs: `message` (C string), `severity` (infoError or fatalError)
- Outputs/Return: None
- Side effects: Calls InitCursor(); displays modal dialog; fatal severity calls ExitToShell()
- Calls: InitCursor, SimpleAlert (Carbon only)
- Notes: **Missing break statement after infoError case** ΓÇö falls through to fatalError.

### alert_user (short, short, short, OSErr)
- Signature: `void alert_user(short severity, short resid, short item, OSErr error)`
- Purpose: Display alert with message from resource (resid, item) and error code.
- Inputs: `severity`, `resid` (string set ID), `item` (string index), `error` (OS error code)
- Outputs/Return: None
- Side effects: Logs to logging system (logError/logFatal); calls InitCursor(); shows dialog; fatal calls ExitToShell()
- Calls: TS_GetCString, sprintf, logError2, logFatal2, SimpleAlert, exit (non-Carbon only)
- Notes: Carbon and Classic Mac paths differ; Classic path uses Alert(129/128).

### vpause (char*)
- Signature: `void vpause(char *message)`
- Purpose: Display warning dialog and log the message.
- Inputs: `message` (C string, max 255 chars)
- Outputs/Return: None
- Side effects: Calls InitCursor(); shows modal dialog; logs warning to logging system.
- Calls: logWarning1, Alert (non-Carbon), SimpleAlert (Carbon)
- Notes: Message is truncated to 255 chars for Classic Mac Pascal string conversion.

### vhalt (char*)
- Signature: `void vhalt(char *message) NORETURN`
- Purpose: Stop recording, log fatal error, show alert, and exit the engine.
- Inputs: `message` (C string)
- Outputs/Return: Never returns (calls ExitToShell)
- Side effects: Calls stop_recording() to clean up recording state; logs fatal; shows modal alert; calls ExitToShell()
- Calls: stop_recording, logFatal1, InitCursor, ExitToShell, Alert (non-Carbon), SimpleAlert (Carbon)
- Notes: Guaranteed termination; used for unrecoverable errors.

### halt (void)
- Signature: `void halt(void) NORETURN`
- Purpose: Immediate halt with debugger break (used for crash-debugging).
- Inputs: None
- Outputs/Return: Never returns
- Side effects: Logs fatal; invokes Debugger(); calls ExitToShell()
- Calls: logFatal, Debugger, ExitToShell
- Notes: Minimal cleanup; used when standard error handling has failed.

### _alephone_assert, _alephone_warn
- Signature: `void _alephone_assert(const char *file, int32 line, const char *what) NORETURN`; `void _alephone_warn(...)`
- Purpose: Handle assertion failures and warnings with file/line context.
- Inputs: Source file, line number, assertion/warning message
- Outputs/Return: _alephone_assert never returns; _alephone_warn returns normally
- Side effects: Format message into assert_text buffer (csprintf); call vhalt or vpause
- Calls: csprintf, vhalt (assert), vpause (warn)
- Notes: Invoked by assert/vassert/warn/vwarn macros defined in csalerts.h (DEBUG builds only).

### SimpleAlert (AlertType, const char*, const char*) [Carbon only]
- Signature: `DialogItemIndex SimpleAlert(AlertType Type, const char *Message, const char *Explain)`
- Purpose: Wrapper around Carbon StandardAlert; converts C strings to Pascal strings.
- Inputs: Alert type, message (C string), explanation (C string, may be NULL)
- Outputs/Return: DialogItemIndex (button pressed)
- Side effects: Calls CopyCStringToPascal (2├ù); calls StandardAlert (modal)
- Calls: CopyCStringToPascal, StandardAlert
- Notes: Only compiled if TARGET_API_MAC_CARBON; used by all alert_user overloads in Carbon builds.

## Control Flow Notes
- **Initialization/Errors**: Called reactively when game encounters errors, failed assertions, or critical conditions.
- **Error Levels**: Info alerts allow continuation; fatal alerts invoke termination.
- **Logging + UI**: All functions log via Logging.h (logError, logWarning, logFatal) in addition to showing dialogs, creating an audit trail.
- **Platform Abstraction**: #ifdef TARGET_API_MAC_CARBON conditionally selects between Carbon (StandardAlert + modern APIs) and Classic Mac (Alert + ParamText).
- **Shutdown**: vhalt/halt ensure stop_recording() is called before termination, preventing recording data loss.

## External Dependencies
- **Headers**: stdlib.h, string.h, Carbon/Carbon.h (conditional), csstrings.h, Logging.h, TextStrings.h
- **Macintosh APIs**: ExitToShell, Debugger, InitCursor, Alert, ParamText, NumToString (Classic); CopyCStringToPascal, StandardAlert (Carbon)
- **Utilities**: TS_GetCString (TextStrings), csprintf (csstrings), getpstr (csstrings), logError/logWarning/logFatal (Logging)
- **External symbols**: stop_recording() (defined elsewhere); ExitToShell marked NORETURN in non-MWERKS builds
