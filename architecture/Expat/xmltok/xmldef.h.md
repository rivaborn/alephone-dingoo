# Expat/xmltok/xmldef.h

## File Purpose
A platform/environment abstraction header that provides conditional memory allocation macros and includes. It allows the Expat XML parser to work with different runtime environments (Windows heap manager, Mozilla NSPR, or standard C library) via compile-time configuration.

## Core Responsibilities
- Conditionally define memory allocation macros (malloc, calloc, free, realloc) based on target environment
- Include platform-specific headers (Windows.h, NSPR headers) when needed
- Provide a transparent abstraction layer so the rest of the codebase uses consistent malloc/free syntax regardless of the actual allocator
- Include string.h globally for string utilities

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. This file contains only macro definitions and conditional includes.

## Control Flow Notes
This is a compile-time configuration header, not runtime code. It is included early in the tokenizer module's build to establish the memory allocation strategy before any tokenization code runs. The actual choice of allocator is determined by preprocessor flags:
- **XML_WINLIB**: Use Windows heap API (HeapAlloc/HeapFree)
- **MOZILLA**: Use NSPR memory functions (PR_Malloc/PR_Free)
- **Default**: Use standard C library (stdlib.h)

## External Dependencies
| Include | Condition | Purpose |
|---------|-----------|---------|
| `string.h` | Always | Standard string utilities |
| `windows.h` | If `XML_WINLIB` defined | Windows heap and API functions |
| `stdlib.h` | If `XML_WINLIB` not defined | Standard C memory/runtime |
| `nspr.h` | If `MOZILLA` defined | Mozilla NSPR memory allocation |

**Macro rewrites** (mallocs, callocs, etc.) conditionally redirect to platform-specific implementations via preprocessor substitution. Note: the `MOZILLA` block also redefines `int` to `int32` for type consistency.
