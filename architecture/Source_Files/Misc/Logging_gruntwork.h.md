# Source_Files/Misc/Logging_gruntwork.h

## File Purpose
Auto-generated convenience header providing logging macros for the game engine. Defines preprocessor macros for quick logging at 8 severity levels with support for main-thread and non-main-thread contexts, variable/fixed argument counts, and scoped log contexts.

## Core Responsibilities
- Provide `logX()` macros (Fatal, Error, Warning, Anomaly, Note, Summary, Trace, Dump) for logging at various severity levels
- Support both main-thread (`logX`) and non-main-thread (`logXNMT`) variants
- Support variable argument count logging via `__VA_ARGS__` (preferred path)
- Support fixed argument counts (1ΓÇô5 args) for compatibility
- Provide `LogContext` macros for RAII-style scoped logging contexts
- Automatically inject `__FILE__` and `__LINE__` metadata into all log calls

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `LogContext` | class (external) | RAII-style object for scoped logging context, initialized by macros |
| Log levels | constants (external) | `logFatalLevel`, `logErrorLevel`, `logWarningLevel`, `logAnomalyLevel`, `logNoteLevel`, `logSummaryLevel`, `logTraceLevel`, `logDumpLevel` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `logDomain` | identifier (external) | file context | Logging domain/category constant passed to logger |

## Key Functions / Methods
All entries are preprocessor macros; no runtime functions defined here.

### logX() / logXNMT() family
- **Purpose**: Main logging macros for 8 severity levels (X Γêê {Fatal, Error, Warning, Anomaly, Note, Summary, Trace, Dump})
- **Variants**:
  - `logX(...)` ΓÇô main-thread, variable args
  - `logXNMT(...)` ΓÇô non-main-thread, variable args
  - `logX1..5(message, arg1..5)` ΓÇô main-thread, fixed 1ΓÇô5 args
  - `logXNMT1..5(...)` ΓÇô non-main-thread, fixed 1ΓÇô5 args
- **Expands to**: `GetCurrentLogger()->logMessage[NMT](logDomain, logXLevel, __FILE__, __LINE__, ...)`
- **Notes**: The `#if 0` block shows obsolete single-arg versions; the `#else` path using `__VA_ARGS__` is preferred and makes numbered variants redundant

### logContext() / logContextNMT() family
- **Purpose**: Create scoped logging contexts (RAII-style)
- **Variants**:
  - `logContext[NMT]0..5(context, arg0..5)`
- **Expands to**: `LogContext makeUniqueIdentifier(_theLogContext,__LINE__)(isNMT, __FILE__, __LINE__, ...)`
- **Side effects**: Declares a temporary `LogContext` object with a unique name (via `makeUniqueIdentifier` macro) to scope context lifetime to the current block
- **Notes**: Creates a new local variable; `makeUniqueIdentifier` prevents name collisions when multiple contexts exist in the same scope

## Control Flow Notes
This file is included utility-wide and is not part of frame/render/update logic. Logging calls are sprinkled throughout the engine for diagnostics. The two code paths (#if 0 / #else) indicate a migration toward variadic macros; the numbered variants exist for backwards compatibility with code that may not support `__VA_ARGS__`.

## External Dependencies
- **GetCurrentLogger()** ΓÇô function returning the active logger object (defined elsewhere)
- **LogContext** ΓÇô class for context-based RAII logging (defined elsewhere)
- **logDomain** ΓÇô domain identifier constant (must be defined in including translation unit)
- **Log level constants** ΓÇô `logFatalLevel`, `logErrorLevel`, etc. (defined elsewhere)
- **makeUniqueIdentifier()** ΓÇô macro for generating unique identifiers (defined elsewhere, likely in a preprocessor utility header)
