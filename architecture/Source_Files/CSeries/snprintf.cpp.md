# Source_Files/CSeries/snprintf.cpp

## File Purpose
Provides platform-agnostic fallback implementations of `snprintf()` and `vsnprintf()` for systems lacking these standard C library functions. Wraps the system's `vsprintf()` with overflow detection and logging warnings.

## Core Responsibilities
- Implement `snprintf()` as a thin variadic wrapper (if `HAVE_SNPRINTF` undefined)
- Implement `vsnprintf()` using `vsprintf()` with buffer-overrun detection (if `HAVE_VSNPRINTF` undefined)
- Emit warnings to the logging system when formatted output exceeds buffer size
- Guard against recursive logging by tracking warning state

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `issuingWarning` | `static bool` | Function-local (vsnprintf) | Prevents recursive logging calls when logging a buffer-overrun warning |

## Key Functions / Methods

### snprintf
- **Signature:** `int snprintf(char* inBuffer, size_t inBufferSize, const char* inFormat, ...)`
- **Purpose:** Variadic wrapper for safe formatted string printing to a fixed-size buffer.
- **Inputs:** Output buffer, buffer size, format string, variable arguments.
- **Outputs/Return:** Number of characters written (from `vsnprintf()`).
- **Side effects:** Writes formatted string to `inBuffer`; delegates to `vsnprintf()`.
- **Calls:** `va_start()`, `vsnprintf()`, `va_end()`.
- **Notes:** Conditional compilationΓÇöonly defined if platform lacks native `snprintf()`. No overflow detection at this level.

### vsnprintf
- **Signature:** `int vsnprintf(char* inBuffer, size_t inBufferSize, const char* inFormat, va_list inArgs)`
- **Purpose:** Core formatted print implementation with overflow detection and warning.
- **Inputs:** Output buffer, buffer size, format string, variadic argument list.
- **Outputs/Return:** Character count from `vsprintf()` (does not truncate output).
- **Side effects:** Writes to buffer via `vsprintf()`; calls `logWarning2()` if overflow detected.
- **Calls:** `vsprintf()`, `logWarning2()` (from Logging.h).
- **Notes:** *Unsafe*ΓÇöuses `vsprintf()` which can overflow the buffer. Detection is post-hoc (after overflow occurs). The `issuingWarning` flag prevents infinite recursion if the warning itself overflows. Deliberately crude and does not truncate output like standard `vsnprintf()` does.

## Control Flow Notes
This is utility infrastructure, not part of the frame loop. Invoked by formatted logging and string operations during normal program execution. Conditional compilation ensures these functions are only compiled on platforms missing native implementations (noted in build configuration checks).

## External Dependencies
- **`<stdio.h>`** ΓÇô Standard I/O (implicitly via `vsprintf()`)
- **`<stdarg.h>`** ΓÇô Variadic argument handling (from snprintf.h)
- **`Logging.h`** ΓÇô `logWarning2()` function for overflow warnings
- **`snprintf.h`** ΓÇô Header with conditional declarations and preprocessor guards
