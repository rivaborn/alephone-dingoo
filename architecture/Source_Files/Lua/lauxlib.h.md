# Source_Files/Lua/lauxlib.h

## File Purpose

Lua auxiliary library header providing convenience functions and macros for building Lua libraries and extending Lua with C code. Defines the public API for type checking, argument validation, library registration, string buffering, and reference management.

## Core Responsibilities

- Define the `luaL_Reg` struct for registering arrays of C functions as Lua libraries
- Provide type-checking and argument-validation helper functions (luaL_check*, luaL_opt*)
- Supply convenience macros for common operations (type tests, conversions, option parsing)
- Implement growable buffer mechanism (`luaL_Buffer`) for efficient string construction
- Define library registration functions (`luaL_register`, `luaI_openlib`)
- Offer error handling utilities with function argument context
- Manage Lua registry references (`luaL_ref`/`luaL_unref`) to prevent garbage collection
- Support code loading from files, buffers, and strings without immediate execution
- Provide backward compatibility for multiple Lua versions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| luaL_Reg | struct | Maps function names to `lua_CFunction` pointers; array passed to `luaL_register` for library setup |
| luaL_Buffer | struct | Growable character buffer with position tracking (`p`), stack level (`lvl`), Lua state reference, and inline buffer |

## Global / File-Static State

None.

## Key Functions / Methods

### luaL_register
- Signature: `void (luaL_register) (lua_State *L, const char *libname, const luaL_Reg *l);`
- Purpose: Register a C library by creating a table and populating it with C functions from a `luaL_Reg` array
- Inputs: `L` (Lua state), `libname` (library namespace), `l` (array of {name, function} pairs)
- Outputs/Return: void; modifies Lua environment/registry
- Side effects: Creates library table, registers all functions, pushes table onto stack
- Calls: `luaI_openlib` (internal)
- Notes: Core mechanism for exposing C function libraries to Lua

### luaL_checktype / luaL_checkany
- Purpose: Validate stack argument type; raise error if mismatch
- Inputs: `L`, `narg` (argument index), `t` (expected type constant like `LUA_TSTRING`)
- Outputs: void; raises error if validation fails
- Side effects: Calls `luaL_typerror` on failure
- Notes: Essential for safe argument handling in C functions

### luaL_checklstring / luaL_checkstring
- Purpose: Extract and validate string from stack, optionally returning length
- Inputs: `L`, `numArg` (stack index), optional `l` (pointer to receive length)
- Outputs: `const char *` to string data
- Side effects: Raises error if not a string; length returned via pointer
- Notes: `luaL_checkstring` macro variant omits length parameter

### luaL_checknumber / luaL_checkinteger
- Purpose: Extract and validate numeric type from stack
- Inputs: `L`, `numArg` (stack index)
- Outputs: `lua_Number` or `lua_Integer` value
- Side effects: Raises error if not convertible to number
- Notes: `luaL_opt*` variants accept default for optional arguments

### luaL_newmetatable
- Signature: `int (luaL_newmetatable) (lua_State *L, const char *tname);`
- Purpose: Create or retrieve metatable for a user-defined type in the registry
- Inputs: `L`, `tname` (type name string)
- Outputs: Returns 1 if newly created, 0 if already existed
- Side effects: Stores/retrieves table in `LUA_REGISTRYINDEX` under `tname`
- Notes: Used to define behavior for userdata objects (e.g., `__index`, `__gc`)

### luaL_checkudata
- Signature: `void *(luaL_checkudata) (lua_State *L, int ud, const char *tname);`
- Purpose: Validate stack value is userdata of expected metatype and extract C pointer
- Inputs: `L`, `ud` (stack index), `tname` (expected type name)
- Outputs: `void *` pointer to userdata storage
- Side effects: Raises error if type mismatch
- Notes: Type safety for C objects wrapped as Lua userdata

### luaL_ref / luaL_unref
- Purpose: Store/retrieve Lua values in registry to prevent garbage collection
- Inputs: `L`, `t` (table index, typically `LUA_REGISTRYINDEX`), `ref` (reference id)
- Outputs: `luaL_ref` returns unique integer ref; `luaL_unref` void
- Side effects: Manages registry entries; `lua_getref` macro retrieves stored value
- Notes: Used when C code must retain references across Lua calls

### luaL_error
- Signature: `int (luaL_error) (lua_State *L, const char *fmt, ...);`
- Purpose: Format and raise error message (printf-style)
- Inputs: `L`, format string, variadic arguments
- Outputs: Does not return (raises error)
- Side effects: Longjmp via Lua error mechanism
- Notes: Alternative to `luaL_argerror` for non-argument errors

### Buffer functions (luaL_buffinit / luaL_prepbuffer / luaL_addlstring / luaL_pushresult)
- Purpose: Efficiently accumulate strings without repeated allocations
- Inputs: `B` (luaL_Buffer *), data to append
- Outputs: `luaL_pushresult` converts buffer to Lua string on stack
- Side effects: Buffer state modified; uses `LUAL_BUFFERSIZE` (static buffer)
- Notes: Macros `luaL_addchar`, `luaL_addsize` for direct buffer manipulation

### File/string loading (luaL_loadfile / luaL_loadbuffer / luaL_loadstring)
- Purpose: Compile Lua source code to bytecode without executing
- Inputs: `L`, source (file path, buffer pointer+size, or string), optional name for debug info
- Outputs: Returns error code (`LUA_ERRSYNTAX`, `LUA_ERRFILE`, 0 on success); leaves chunk on stack
- Side effects: Loads bytecode onto stack for later execution via `lua_call` or `lua_pcall`
- Notes: `LUA_ERRFILE` is auxiliary-specific error for file I/O failures

### Utility functions (luaL_where / luaL_gsub / luaL_findtable / luaL_checkoption)
- `luaL_where`: Push debug location string (file:line) to stack for error context
- `luaL_gsub`: String substitution (analog of Lua `string.gsub`)
- `luaL_findtable`: Navigate/create nested table hierarchy
- `luaL_checkoption`: Match argument against array of string options, return index

## Convenience Macros

- **Type conversions**: `luaL_checkint`, `luaL_checklong`, `luaL_optint`, `luaL_optlong` (cast integer variants)
- **Type querying**: `luaL_typename(L,i)` ΓåÆ type name as string
- **Code execution**: `luaL_dofile`, `luaL_dostring` ΓåÆ load and execute in one call
- **Metatable access**: `luaL_getmetatable(L,n)` ΓåÆ push metatable from registry
- **Argument checking**: `luaL_argcheck(L, cond, numarg, msg)` ΓåÆ conditional error

## Control Flow Notes

Header-only; no control flow. Defines C API contract for extending Lua. Typical usage:
1. C code implements `lua_CFunction` callbacks
2. Array of `luaL_Reg` entries maps Lua function names to C implementations
3. `luaL_register` exposes library to Lua environment
4. C functions validate arguments via `luaL_check*` family
5. C functions manipulate Lua stack via lua.h core API
6. C functions return argument count (0 or positive integer)

## External Dependencies

- `lua.h`: Core Lua C API (defines `lua_State`, `lua_CFunction`, `LUA_REGISTRYINDEX`, `LUA_MULTRET`, error codes)
- `config.h`: Build configuration flag `HAVE_LUA`
- Standard C: `<stddef.h>` (size_t), `<stdio.h>` (FILE)
