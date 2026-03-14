# Source_Files/CSeries/csalerts_sdl.cpp

## File Purpose
SDL implementation of game alerts, assertions, and debugging support. Displays user-facing error/warning dialogs or console output depending on video availability, and provides assertion/halt handlers for the game engine.

## Core Responsibilities
- Display formatted alert dialogs (warnings/errors) with text wrapping to users
- Log alerts to the game logging system alongside user display
- Handle assertion failures with file/line context
- Provide pause/halt/debug breakpoint entry points
- Gracefully degrade to stderr output when SDL video is unavailable
- Manage program termination for fatal errors

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `dialog` | class (from sdl_dialogs.h) | Modal dialog container for alert message UI |
| `vertical_placer` | class (from sdl_dialogs.h) | Layout manager for dialog widgets |
| `w_title`, `w_static_text`, `w_button` | widget classes (from sdl_widgets.h) | UI components for alert title, message text, OK/QUIT button |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `assert_text` | `char[256]` | static | Buffer for formatting assertion messages with file/line info |
| `MAX_ALERT_WIDTH` | `const int` | file-static | Max pixel width for alert message text before wrapping |

## Key Functions / Methods

### alert_user (first overload)
- **Signature:** `void alert_user(const char *message, short severity)`
- **Purpose:** Display a styled alert dialog to the user, or stderr if no video surface available.
- **Inputs:** Message text; severity flag (`infoError` for warning, `fatalError` for error)
- **Outputs/Return:** None (dialog blocks until user clicks OK/QUIT)
- **Side effects:** Creates/runs modal dialog; calls `update_game_window()` after dialog closes (info-level only); logs via Logging system; calls `exit(1)` for fatal severity
- **Calls:** `SDL_GetVideoSurface()`, `get_theme_font()`, `text_width()`, `logError2()`, `logFatal2()`, `update_game_window()`
- **Notes:** Implements word-wrapping at `MAX_ALERT_WIDTH`; allocates widget hierarchy that dialog owns; frees temp `strdup()` buffer

### alert_user (second overload)
- **Signature:** `void alert_user(short severity, short resid, short item, OSErr error)`
- **Purpose:** Display a resource-based alert message with error code context.
- **Inputs:** Severity flag; resource ID and item ID for message lookup; OS error code
- **Outputs/Return:** None
- **Side effects:** Logs via `logError2()` or `logFatal2()`; delegates to first `alert_user()` overload
- **Calls:** `getcstr()`, `sprintf()`, `logError2()`, `logFatal2()`, `alert_user(const char*, short)`
- **Notes:** Constructs message as "resource_string (error N)"

### pause_debug
- **Signature:** `void pause_debug(void)`
- **Purpose:** Breakpoint entry point for debugger; logs and writes to stderr.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Logs via `logNote()`; prints "pause\n" to stderr
- **Calls:** `logNote()`, `fprintf()`
- **Notes:** Currently a stub; comment suggests it should invoke a debugger

### vpause
- **Signature:** `void vpause(const char *message)`
- **Purpose:** Log and display a warning message without halting.
- **Inputs:** Message text
- **Outputs/Return:** None
- **Side effects:** Logs via `logWarning1()`; prints to stderr
- **Calls:** `logWarning1()`, `fprintf()`

### halt
- **Signature:** `void halt(void)`
- **Purpose:** Unconditional program termination.
- **Inputs:** None
- **Outputs/Return:** None (never returns)
- **Side effects:** Logs via `logFatal()`; prints "halt\n" to stderr; calls `abort()`
- **Calls:** `logFatal()`, `fprintf()`, `abort()`

### vhalt
- **Signature:** `void vhalt(const char *message)`
- **Purpose:** Display a fatal message and halt the program.
- **Inputs:** Message text
- **Outputs/Return:** None (never returns)
- **Side effects:** Calls `stop_recording()`; logs via `logFatal1()`; flushes logger; prints to stderr; calls `abort()`
- **Calls:** `stop_recording()`, `logFatal1()`, `GetCurrentLogger()->flush()`, `fprintf()`, `abort()`

### _alephone_assert
- **Signature:** `void _alephone_assert(const char *file, int32 line, const char *what)`
- **Purpose:** Assertion failure handler for compile-time macro support.
- **Inputs:** Source file path, line number, assertion condition/message
- **Outputs/Return:** None (never returns)
- **Side effects:** Formats assertion text with file:line context; calls `vhalt()` to exit
- **Calls:** `csprintf()`, `vhalt()`

### _alephone_warn
- **Signature:** `void _alephone_warn(const char *file, int32 line, const char *what)`
- **Purpose:** Assertion warning handler (non-fatal variant).
- **Inputs:** Source file path, line number, warning condition/message
- **Outputs/Return:** None
- **Side effects:** Formats message with file:line; calls `vpause()`
- **Calls:** `csprintf()`, `vpause()`

## Control Flow Notes
This file provides the final-stage error/alert handling for the game engine. Execution reaches here from assertion macros or explicit error calls during gameplay. The `alert_user()` functions are entry points for normal (non-fatal) messages; they preserve game state and return control. `vhalt()` and `halt()` are terminal paths that clean up recording, log, and exit. The file assumes the main game loop has already initialized SDL and logging.

## External Dependencies
- **SDL headers:** `<SDL.h>` (via cseries.h); provides video surface, dialogs, events
- **Logging system:** `Logging.h`; exports `logFatal()`, `logError2()`, `logWarning1()`, `logNote()`, `GetCurrentLogger()`, `logFatal1()`
- **Dialog/widget system:** `sdl_dialogs.h`, `sdl_widgets.h`; exports `dialog`, `vertical_placer`, `w_title`, `w_static_text`, `w_spacer`, `w_button`, `get_theme_font()`, `text_width()`, `dialog_ok`
- **Game engine:** `update_game_window()` (defined elsewhere), `stop_recording()` (defined elsewhere)
- **Utility:** `cseries.h` (master header); `<stdio.h>`, `<stdlib.h>` (abort, sprintf)
- **Defined elsewhere:** `getcstr()`, `csprintf()`, `font_info`, `MESSAGE_WIDGET`, `LABEL_WIDGET`, `TITLE_WIDGET`, `SPACER_WIDGET`, `BUTTON_WIDGET`, `dialog_ok` callback
