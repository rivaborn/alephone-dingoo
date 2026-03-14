# Source_Files/Misc/Logging.h

## File Purpose
Defines a flexible, hierarchical logging system with severity levels, context stacks, and configurable output. Provides abstract Logger base class and convenience macros for logging at various levels (fatal, error, warning, anomaly, note, summary, trace, dump) with variadic arguments and thread-safety variants.

## Core Responsibilities
- Define 8 severity levels for categorizing messages (logFatalLevel=0 through logDumpLevel=60)
- Declare abstract `Logger` base class for pluggable backend implementations
- Provide convenience macros (`logError()`, `logWarning()`, etc.) wrapping logger calls with file/line info
- Manage per-domain logging configuration (threshold level, location display, auto-flush behavior)
- Support stack-based context nesting via `LogContext` class for hierarchical logging
- Parse logging configuration via XML backend (through `Logging_GetParser()`)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Logger | abstract class | Interface for logging backends; subclasses implement actual message/context handling |
| LogContext | class | RAII-style scope guard for entering/leaving hierarchical logging contexts; stores contextSet flag and nonMainThread hint |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| logDomain | `extern const char*` | global | Default domain for log messages when caller doesn't specify |

## Key Functions / Methods

### Logger::logMessage
- Signature: `virtual void logMessage(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, ...)`
- Purpose: Main-thread logging with variadic printf-style arguments
- Inputs: domain name, severity level, source file, line number, format string, args
- Outputs/Return: void
- Side effects: Writes to logging backend (implementation-defined)
- Calls: Delegates to `logMessageV()` after va_start
- Notes: Non-virtual implementation wraps to variadic version; main thread only

### Logger::logMessageNMT
- Signature: `virtual void logMessageNMT(const char* inDomain, int inLevel, const char* inFile, int inLine, const char* inMessage, ...)`
- Purpose: Non-main-thread variant with possibly different or missing implementation for thread safety
- Inputs: Same as `logMessage`
- Outputs/Return: void
- Side effects: Logging output (thread-safe implementation expected in subclass)
- Calls: Delegates to `logMessageV()` via va_start

### Logger::pushLogContext / pushLogContextV
- Signature: `virtual void pushLogContext(const char* inFile, int inLine, const char* inContext, ...)`
- Purpose: Enter a named logging context (for hierarchical log output)
- Inputs: source location, context name, optional args for printf-style formatting
- Outputs/Return: void
- Side effects: Updates internal context stack in logger
- Calls: Delegates to `pushLogContextV()` via va_list

### Logger::popLogContext
- Signature: `virtual void popLogContext() = 0`
- Purpose: Exit current logging context
- Outputs/Return: void
- Side effects: Pops context stack

### Logger::flush
- Signature: `virtual void flush() = 0`
- Purpose: Flush pending output (e.g., buffered messages to disk)
- Outputs/Return: void

### GetCurrentLogger
- Signature: `Logger* GetCurrentLogger()`
- Purpose: Retrieve the active logger instance (singleton-style access point)
- Outputs/Return: Pointer to active Logger
- Calls: Used by all macros to dispatch logging calls

### LogContext constructors / methods
- Overloaded constructors allow inline declaration: `LogContext(const char* inFile, int inLine, const char* inContext, ...)`
- `enterContext()` / `enterContextV()` manually push a context
- `leaveContext()` manually pop; also called in destructor for RAII cleanup
- Notes: On classic Mac OS (non-OSX), non-main-thread contexts are no-ops (guarded by `#if defined(mac) && !defined(__MACH__)`)

## Control Flow Notes
- **Initialization**: `GetCurrentLogger()` must be called to obtain logger; domains can be configured via `setLoggingThreshhold()`, `setShowLoggingLocations()`, `setFlushLoggingOutput()`
- **Message logging**: All convenience macros (`logError()`, `logWarning()`, etc.) invoke `GetCurrentLogger()->logMessage()` with `__FILE__` and `__LINE__` automatically injected
- **Context stacking**: `logContext("...")` (or `logContextNMT("...")` for threads) declares a temporary `LogContext` object on the stack; destructor pops on scope exit
- **Thread variants**: NMT (non-main-thread) macros exist for all log levels and context operations; implementation may be simpler or safer
- Not tied to a specific frame/update cycle; used opportunistically throughout engine code

## External Dependencies
- `<stdarg.h>` ΓÇö variadic argument handling
- `XML_ElementParser` (defined elsewhere) ΓÇö for parsing logging config via `Logging_GetParser()`
- Platform check: `#if defined(mac) && !defined(__MACH__)` ΓÇö special handling for classic Mac OS (OSX paths this code)
- Logging_gruntwork.h (auto-generated) ΓÇö provides all convenience macros (`logError()`, `logWarning1()`, `logContext2()`, etc.)

---

**Notes**: 
- The file header incorrectly says "Logging.cpp" but this is a `.h` file; likely copy-paste from implementation.
- Commented-out `SubLogger` class sketches future "deferred logging" feature (log to temporary buffer, optionally promote to main log).
- Commented-out `logCheckWarn/logCheckAnomaly` macros were attempted but disabled; condition-testing macros not currently exposed.
