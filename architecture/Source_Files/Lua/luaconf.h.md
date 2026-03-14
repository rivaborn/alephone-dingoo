# Source_Files/Lua/luaconf.h

## File Purpose
Configuration header for Lua interpreter/VM. Defines platform-specific preprocessor macros that customize Lua's behavior, type system, stack limits, and module loading paths without modifying core source files.

## Core Responsibilities
- Platform detection (Windows, POSIX/Linux, macOS) and feature flag setup
- Module and C library search paths (`LUA_PATH`, `LUA_CPATH`)
- Type configuration (integer width, number format, memory types)
- API symbol visibility and calling convention (`LUA_API`, `LUALIB_API`)
- Stack/recursion limits and array bounds (`LUAI_MAXCALLS`, `LUAI_MAXVARS`, etc.)
- Garbage collector tuning (`LUAI_GCPAUSE`, `LUAI_GCMUL`)
- Numeric operations (double-precision arithmetic macros)
- Dynamic library loading method selection
- Exception handling mechanism (C++/longjmp/setjmp)
- Backward compatibility flags for deprecated Lua 5.0 features

## Key Types / Data Structures
NoneΓÇöfile contains only macro definitions. Referenced types configured by macros:

| Name | Kind | Purpose |
|------|------|---------|
| `lua_Integer` | typedef (ΓåÆ `ptrdiff_t`) | Integral type for Lua integers |
| `lua_Number` | typedef (ΓåÆ `double`) | Numeric type for Lua values |
| `LUAI_UINT32` / `LUAI_INT32` | typedef | Platform-independent 32-bit integers |
| `LUAI_UMEM` / `LUAI_MEM` | typedef | Memory accounting types |
| `luai_jmpbuf` | typedef | Exception handling buffer (platform-dependent) |

## Global / File-Static State
None.

## Key Functions / Methods
NoneΓÇöthis is a pure configuration header with no function definitions.

## Control Flow Notes
Purely compile-time; no runtime control flow. Conditional compilation based on:
- `_WIN32` / `LUA_WIN` (Windows platform)
- `LUA_USE_POSIX`, `LUA_USE_LINUX`, `LUA_USE_MACOSX` (Unix variants)
- `LUA_ANSI`, `__STRICT_ANSI__` (C standard compliance)
- `LUA_CORE`, `LUA_LIB` (DLL export control)
- `LUA_BUILD_AS_DLL`, `__GNUC__`, `__ELF__` (visibility/ABI)

Selects error handling via `__cplusplus` (C++ exceptions) vs. `LUA_USE_ULONGJMP` (Unix) vs. default `longjmp`.

## External Dependencies
- Standard C: `<limits.h>`, `<stddef.h>`, `<math.h>`, `<assert.h>`, `<unistd.h>`, `<io.h>`, `<stdio.h>`
- Optional readline: `<readline/readline.h>`, `<readline/history.h>` (if `LUA_USE_READLINE`)
- Generated config: `config.h` (from bundled context, defines `HAVE_OPENGL`, `HAVE_SDL_IMAGE`, `HAVE_SDL_NET`, etc.)
- Platform-specific symbols (not included): Windows `_isatty()`, `_fileno()`, `_popen()`; Unix `isatty()`, `popen()`, `mkstemp()`, `dlopen()`
