# Source_Files/XML/XML_DataBlock.cpp

## File Purpose
Implements the `XML_DataBlock` class, a memory-based XML parser adapter that handles data block parsing and provides platform-specific error reporting. Manages read errors, XML parse errors, and interpretation errors with different handling for macOS (Classic/Carbon) and SDL platforms.

## Core Responsibilities
- Retrieves and validates XML data from in-memory buffers
- Reports read/IO errors with platform-specific alert dialogs (macOS) or logging (SDL)
- Reports XML parsing errors with line numbers and source identification for debugging
- Reports interpretation errors with throttling to prevent log spam (max 7 errors)
- Requests parsing abort when error count exceeds threshold
- Maintains source name context for clearer error messages

## Key Types / Data Structures
None (implements inherited `XML_DataBlock` class; see header).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MaxErrorsToShow` | const int | file-static | Caps interpretation error messages at 7 to avoid flooding |
| `FatalErrorAlert` | const short | file-static (mac) | macOS Classic alert dialog resource ID (128) |
| `NonFatalErrorAlert` | const short | file-static (mac) | macOS Classic alert dialog resource ID (129) |

## Key Functions / Methods

### GetData
- **Signature:** `bool GetData()`
- **Purpose:** Signals to base class that XML data is ready to parse; validates buffer state.
- **Inputs:** None (uses inherited `Buffer` and `BufLen` members).
- **Outputs/Return:** Always returns `true`.
- **Side effects:** Sets `LastOne = true` to indicate this is the only/final data block.
- **Calls:** `assert()`.
- **Notes:** Minimal implementation; actual parsing logic resides in base class `XML_Configure::DoParse()`.

### ReportReadError
- **Signature:** `void ReportReadError()`
- **Purpose:** Report I/O or resource read failure with source identification.
- **Inputs:** None (uses inherited `SourceName` member).
- **Outputs/Return:** None (displays alert, then exits program).
- **Side effects:** Calls platform-specific alert/dialog functions; terminates program via `ExitToShell()` (macOS) or `exit(1)` (SDL).
- **Calls:** `csprintf()`, `psprintf()`, `SimpleAlert()`, `ParamText()`, `Alert()`, `ExitToShell()`, `fprintf()`, `exit()`.
- **Notes:** No error recovery; always fatal. macOS Classic and Carbon APIs differ.

### ReportParseError
- **Signature:** `void ReportParseError(const char *ErrorString, int LineNumber)`
- **Purpose:** Report XML syntax error with line number and source identification.
- **Inputs:** `ErrorString` (error description), `LineNumber` (line where error occurred).
- **Outputs/Return:** None (displays alert, then exits program on macOS; logs on SDL).
- **Side effects:** Fatal on macOS; logged but non-fatal on SDL platforms.
- **Calls:** `csprintf()`, `psprintf()`, `SimpleAlert()`, `ParamText()`, `Alert()`, `ExitToShell()`, `fprintf()`.
- **Notes:** Includes `SourceName` in output for disambiguation when multiple XML sources are in use.

### ReportInterpretError
- **Signature:** `void ReportInterpretError(const char *ErrorString)`
- **Purpose:** Report semantic/interpretation errors during XML processing with throttling to prevent log flooding.
- **Inputs:** `ErrorString` (error description).
- **Outputs/Return:** None (logs via Logging system).
- **Side effects:** Logs to system logger; once 7 errors shown, subsequent errors are suppressed with a "(more errors not shown)" message.
- **Calls:** `GetNumInterpretErrors()` (inherited), `logAnomaly1()`, `logAnomaly()`.
- **Notes:** Non-fatal; parsing continues. Unlike parse errors, interpretation errors are recoverable.

### RequestAbort
- **Signature:** `bool RequestAbort()`
- **Purpose:** Check if accumulated errors warrant aborting the parse to prevent further cascading errors.
- **Inputs:** None (uses inherited error count).
- **Outputs/Return:** Returns `true` if error count >= 7; `false` otherwise.
- **Side effects:** None.
- **Calls:** `GetNumInterpretErrors()` (inherited).
- **Notes:** Parser can call this after each interpretation error to decide whether to continue.

## Control Flow Notes
This class participates in the XML parsing lifecycle as a "data source" for the base class `XML_Configure`:
1. **Init:** `ParseData()` (inherited public method) copies buffer pointers and calls `DoParse()` in base.
2. **Parse:** Base class calls `GetData()` repeatedly; this implementation signals all data is ready (`LastOne=true`).
3. **Error handling:** During base-class parsing, parse/read/interpretation errors trigger the three error-reporting methods.
4. **Abort:** Base class may call `RequestAbort()` to decide whether to continue after interpretation errors.

## External Dependencies
- **Includes:** `<string.h>` (C standard), `cseries.h` (platform abstraction layer), `Logging.h` (logging macros).
- **Inherited:** `XML_Configure` (base class with `DoParse()`, `GetNumInterpretErrors()`, `Buffer`, `BufLen`).
- **External symbols:** Platform-specific APIs (`ExitToShell`, `SimpleAlert`, `ParamText`, `Alert` on macOS; `fprintf`, `exit` on SDL); logging functions (`logAnomaly1`, `logAnomaly`); cseries functions (`csprintf`, `psprintf`).
