# Source_Files/Misc/Logging.cpp

## File Purpose
Implements a flexible, hierarchical logging system for the Aleph One game engine. Provides context-aware message routing to a platform-specific log file with configurable thresholds, indentation, and flushing behavior. Supports XML-based runtime configuration.

## Core Responsibilities
- Maintains a singleton Logger instance with lazy initialization
- Manages a context stack for hierarchical log indentation ("while X, while Y, message")
- Filters and routes log messages to file based on level threshold
- Supports domain-based logging (infrastructure in place, not yet utilized)
- Parses XML configuration to dynamically adjust logging settings
- Handles platform-specific log file paths (Linux `~/.alephone/`, macOS `~/Library/Logs/`)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Logger` | class (abstract) | Base interface for logging operations; defined in header |
| `TopLevelLogger` | class | Concrete logger with context stack tracking and message formatting |
| `XML_LoggingConfigurationParser` | class | Parses `<logging_domain>` XML elements for runtime config |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sCurrentLogger` | `Logger*` | static | Singleton logger instance |
| `sOutputFile` | `FILE*` | static | Open file handle for log output |
| `sLoggingThreshhold` | `int` | static | Messages below this level are logged (inverse threshold) |
| `sShowLocations` | `bool` | static | Whether to append `(file:line)` to log lines |
| `sFlushOutput` | `bool` | static | Whether to flush file after each write (safety vs. perf) |
| `logDomain` | `const char*` | extern | Default domain identifier ("global") |

## Key Functions / Methods

### GetCurrentLogger
- **Signature:** `Logger* GetCurrentLogger()`
- **Purpose:** Returns the active logger, initializing it on first call
- **Inputs:** None
- **Outputs/Return:** Pointer to singleton `TopLevelLogger`
- **Side effects:** Calls `InitializeLogging()` once to create log file and logger
- **Calls:** `InitializeLogging()` (lazy init)
- **Notes:** Acts as singleton factory; safe for repeated calls

### Logger::pushLogContext
- **Signature:** `void pushLogContext(const char* inFile, int inLine, const char* inContext, ...)`
- **Purpose:** Variadic convenience wrapper entering a named logging context
- **Inputs:** File, line, format string with varargs
- **Outputs/Return:** None; calls virtual `pushLogContextV`
- **Side effects:** Adds formatted context string to the logger's context stack
- **Calls:** `pushLogContextV()` (virtual, implemented by subclass)
- **Notes:** For main-thread use only

### Logger::logMessage
- **Signature:** `void logMessage(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, ...)`
- **Purpose:** Variadic convenience wrapper for logging a message
- **Inputs:** Domain, severity level, file, line, format string with varargs
- **Outputs/Return:** None; calls virtual `logMessageV`
- **Side effects:** Routes message to file if level is below threshold
- **Calls:** `logMessageV()` (virtual, implemented by subclass)
- **Notes:** For main-thread use only

### Logger::logMessageNMT
- **Signature:** `void logMessageNMT(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, ...)`
- **Purpose:** Non-main-thread-safe variant of `logMessage`
- **Inputs:** Same as `logMessage`
- **Outputs/Return:** None
- **Side effects:** Empty on Mac OS 9 (crashes there); otherwise same as `logMessage`
- **Calls:** `logMessageV()` (conditionally)
- **Notes:** NMT = non-main-thread; disabled on old Mac for stability

### TopLevelLogger::pushLogContextV
- **Signature:** `void pushLogContextV(const char* inFile, int inLine, const char* inContext, va_list inArgs)`
- **Purpose:** Formats a context string and pushes it onto the context stack
- **Inputs:** File, line, format string, va_list args
- **Outputs/Return:** None
- **Side effects:** Appends formatted string (and optional file:line) to `mContextStack`
- **Calls:** `vsnprintf()`, `snprintf()`
- **Notes:** If `sShowLocations` is true, appends `(file:line)` to context. Choice deferred until log time (comment notes this limitation).

### TopLevelLogger::popLogContext
- **Signature:** `void popLogContext()`
- **Purpose:** Removes the most recent context from the stack
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Pops `mContextStack`; updates `mMostRecentCommonStackDepth` if stack shrinks
- **Calls:** None
- **Notes:** Tracks "common" stack depth to avoid re-printing contexts unnecessarily

### TopLevelLogger::logMessageV
- **Signature:** `void logMessageV(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, va_list inArgs)`
- **Purpose:** Core logging routine; formats and writes message with context indentation
- **Inputs:** Domain, level, file, line, format string, va_list args
- **Outputs/Return:** None
- **Side effects:** Writes to `sOutputFile` if open and level < threshold; conditionally flushes
- **Calls:** `vsnprintf()`, `snprintf()`, `fprintf()`, `fflush()`
- **Notes:** Indents output by 2 spaces per context depth. Prints any new context lines above the message. Domains currently unused (infrastructure ready for future multi-domain routing).

### InitializeLogging
- **Signature:** `static void InitializeLogging()`
- **Purpose:** One-time setup: open log file and create logger instance
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Opens `sOutputFile` with append mode; creates new `TopLevelLogger`; writes timestamp header
- **Calls:** `getenv()`, `getpwuid()`, `fopen()`, `time()`, `ctime()`, `fprintf()`
- **Notes:** Platform-specific paths (Unix/macOS vs. fallback). Asserts `sOutputFile == NULL` to catch double-init.

### Logging_GetParser
- **Signature:** `XML_ElementParser* Logging_GetParser()`
- **Purpose:** Returns the root XML parser for logging configuration
- **Inputs:** None
- **Outputs/Return:** Pointer to static `LoggingParser` with `LoggingConfigurationParser` as child
- **Side effects:** Registers child parser on first call
- **Calls:** `AddChild()`
- **Notes:** Used during engine startup to parse XML config sections

## Control Flow Notes
- **Initialization:** Lazy singletonΓÇöfirst call to `GetCurrentLogger()` triggers `InitializeLogging()`, which opens log file and creates logger.
- **Per-message flow:** Log calls check threshold, format context lines (if not already printed), format message, write to file, optionally flush.
- **Context lifecycle:** Context pushed on scope entry (via macro or `LogContext` RAII class), popped on scope exit. Output is indented per depth.
- **Configuration:** XML parser (`XML_LoggingConfigurationParser`) reads domain/threshold/location/flush settings and applies them via static setter functions.

## External Dependencies
- **Standard library:** `<fstream>`, `<string>`, `<vector>`, `<time.h>`, `<stdio.h>`, `<stdarg.h>`
- **Platform-specific (Unix/POSIX):** `<unistd.h>`, `<sys/types.h>`, `<pwd.h>` for home directory lookup
- **Project headers:**
  - `Logging.h` ΓÇö class definitions
  - `cseries.h` ΓÇö platform defs, macros
  - `snprintf.h` ΓÇö portable `vsnprintf()`/`snprintf()` (fallback)
  - `XML_ElementParser.h` ΓÇö XML parsing base class and helpers (`StringsEqual`, `ReadInt16Value`, `ReadBooleanValueAsBool`)
