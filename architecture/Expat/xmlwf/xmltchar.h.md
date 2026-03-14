# Expat/xmlwf/xmltchar.h

## File Purpose
A portability header providing compile-time abstraction over wide-character (UTF-16) vs. narrow-character (ASCII) text APIs. It uses preprocessor macros to unify string and I/O function names, allowing single-source compilation for both Unicode and non-Unicode builds of the xmlwf tool.

## Core Responsibilities
- Abstract character width differences via macro aliasing
- Provide generic text function names (T, ftprintf, tcscmp, etc.)
- Enforce 16-bit Unicode wchar_t constraint when XML_UNICODE is active
- Map generic names to platform-specific wide-character functions (_wfopen, fwprintf, wmain, etc.)
- Support fallback to standard C library functions when Unicode is disabled

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. (Entirely preprocessor macro definitions.)

## Macro Mappings

**When `XML_UNICODE` is defined:**

| Macro | Maps To | Purpose |
|-------|---------|---------|
| T(x) | L##x | Wide-character string literal prefix |
| ftprintf | fwprintf | Wide formatted output |
| tfopen | _wfopen | Wide file open |
| tcscmp, tcscpy, tcscat, tcschr, tcsrchr, tcslen | wcscmp, wcscpy, wcscat, wcschr, wcsrchr, wcslen | Wide string operations |
| tmain | wmain | Wide-character main entry point |
| topen, tremove, tperror | _wopen, _wremove, _wperror | Wide system calls |

**When `XML_UNICODE` is NOT defined:**

All `t*` macros map to standard C library equivalents (strcmp, strcpy, fprintf, fopen, main, etc.).

## Control Flow Notes
This is a header-only configuration file included early in compilation to establish consistent naming conventions throughout xmlwf. No runtime control flow.

## External Dependencies
- Standard C library (stdio.h, string.h, stdlib.h implied)
- Platform-specific: Windows wide-character API functions (_wfopen, _wopen, wmain, etc.)

## Notes
- The `#error` directive enforces that `XML_UNICODE` requires 16-bit `wchar_t` (some platforms default to 32-bit), preventing silent miscompilation.
- Classic C preprocessor portability pattern; avoids duplicating function call sites.
