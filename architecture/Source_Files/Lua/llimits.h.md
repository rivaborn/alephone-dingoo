# Source_Files/Lua/llimits.h

## File Purpose
This header defines Lua VM internal limits, platform-specific type aliases, and utility macros for the Lua 5.1 runtime. It establishes maximum constraints for stack depth, memory allocation, and string tables, plus helper macros for type casting, assertions, and thread synchronization used throughout the Lua engine.

## Core Responsibilities
- Define portable integer types (lu_int32, lu_mem, l_mem, lu_byte) abstracting platform differences
- Establish safe upper bounds for stack (MAXSTACK=250), memory (MAX_LUMEM, MAX_INT), and string tables
- Provide casting macros (cast, cast_byte, cast_num) and assertion helpers for type safety and debugging
- Define threading primitives (lua_lock/lua_unlock) for optional synchronization
- Declare Instruction type for VM bytecode
- Suppress compiler warnings via UNUSED() macro for unused parameters

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| lu_int32 | typedef | 32-bit unsigned integer (via LUAI_UINT32) |
| lu_mem | typedef | Unsigned memory size type (via LUAI_UMEM) |
| l_mem | typedef | Signed memory type for memory management (via LUAI_MEM) |
| lu_byte | typedef | Unsigned char for non-character small values |
| L_Umaxalign | typedef | Alignment type ensuring max struct padding (via LUAI_USER_ALIGNMENT_T) |
| l_uacNumber | typedef | Number type after usual argument conversion (via LUAI_UACNUMBER) |
| Instruction | typedef | VM bytecode instruction (lu_int32, ΓëÑ4 bytes) |

## Global / File-Static State
None.

## Key Functions / Methods
All are macros; no runtime functions.

### Limit Constants
- **MAX_SIZET**: Maximum size_t minus 2 (safety margin to prevent wraparound)
- **MAX_LUMEM**: Maximum lu_mem minus 2 for safety
- **MAX_INT**: INT_MAX minus 2 to avoid edge cases
- **MAXSTACK**: 250 ΓÇö max call stack depth per Lua function
- **MINSTRTABSIZE**: 32 ΓÇö minimum string intern table size (power of 2)
- **LUA_MINBUFFER**: 32 ΓÇö minimum string buffer size

### Type Casting Macros
- **cast(t, exp)**: Generic C-style cast `(t)(exp)`
- **cast_byte(i), cast_num(i), cast_int(i)**: Specialized casts to lu_byte, lua_Number, int

### Debug/Assertion Macros
- **lua_assert(c)**: Conditional assertion (no-op unless overridden in luaconf)
- **check_exp(c,e)**: Assert condition *c*, evaluate and return *e*
- **api_check(l,e)**: API boundary assertion (calls luai_apicheck or lua_assert)

### Threading Macros
- **lua_lock(L), lua_unlock(L)**: Thread synchronization (default: no-op; can be overridden for mutex)
- **luai_threadyield(L)**: Yield thread by unlocking then relocking

### Utility Macros
- **IntPoint(p)**: Hash a pointer to unsigned int via `(unsigned int)(lu_mem)(p)`
- **UNUSED(x)**: Suppress unused-parameter warnings via `(void)(x)`
- **condhardstacktests(x)**: Execute *x* only if HARDSTACKTESTS defined (debug helper)

## Control Flow Notes
No control flow ΓÇö pure header with type definitions and constants resolved at compile time. Used by all Lua VM implementation files.

## External Dependencies
- `<limits.h>` ΓÇö C standard integer limits (INT_MAX)
- `<stddef.h>` ΓÇö Standard types (size_t, NULL)
- `lua.h` ΓÇö Public Lua API (transitively pulls luaconf.h for LUAI_* macros)
- **Symbols expected from luaconf.h (defined elsewhere):** LUAI_UINT32, LUAI_UMEM, LUAI_MEM, LUAI_USER_ALIGNMENT_T, LUAI_UACNUMBER, optional lua_assert override
- **config.h guard:** Conditional compilation (HAVE_LUA) for Aleph One integration
