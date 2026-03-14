# Source_Files/Lua/lualib.h

## File Purpose
Header declaring Lua standard library initialization functions for the game engine's scripting system. Provides function declarations to open/register Lua's built-in modules (base, table, I/O, OS, string, math, debug, package) into a Lua state. Wrapped in `HAVE_LUA` guard, indicating scripting is an optional feature.

## Core Responsibilities
- Declare library initialization functions (`luaopen_*`) for each Lua standard library module
- Define symbolic names (macros) for library identifiers used in registration
- Declare `luaL_openlibs()` as a convenience function to load all libraries at once
- Define file-handle type key for Lua's I/O library
- Provide a default `lua_assert` macro if not already defined by the host

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### luaopen_base
- Signature: `int (luaopen_base)(lua_State *L)`
- Purpose: Initialize and register Lua's base library (print, assert, type, etc.)
- Inputs: Lua state pointer
- Outputs/Return: Status code (implementation-dependent, typically 0 or 1)
- Side effects: Modifies Lua state by adding base library functions to global namespace
- Calls: Not visible in this header (implementation in lualib.c)
- Notes: Called during Lua VM setup; one of the core libraries

### luaopen_table, luaopen_io, luaopen_os, luaopen_string, luaopen_math, luaopen_debug, luaopen_package
- Signature: `int (luaopen_*)(lua_State *L)` (pattern repeats for each library)
- Purpose: Initialize and register each respective Lua standard library module
- Inputs: Lua state pointer
- Outputs/Return: Status code
- Side effects: Adds module functions to Lua state
- Calls: Not visible in this header
- Notes: Each library provides a separate module namespace; individually optional in initialization

### luaL_openlibs
- Signature: `void (luaL_openlibs)(lua_State *L)`
- Purpose: Convenience function that opens all standard libraries at once
- Inputs: Lua state pointer
- Outputs/Return: void
- Side effects: Calls all `luaopen_*` functions to populate the Lua environment
- Calls: Not visible in this header (calls all luaopen_* functions internally)
- Notes: Common entry point for full standard library initialization; simplifies setup

## Control Flow Notes
This is a pure declaration header with no implementation. Control flow depends on where/when the game engine calls these functions:
- Typically invoked during Lua VM initialization (game startup or when loading scripts)
- Either call `luaL_openlibs()` for full setup, or selectively call individual `luaopen_*()` functions for minimal overhead
- The conditional compilation via `HAVE_LUA` means the entire Lua subsystem can be disabled at build time

## External Dependencies
- **Includes**: `config.h` (build-time feature flags), `lua.h` (core Lua C API, defines `lua_State`)
- **External symbols**: `lua_State` (opaque Lua execution context, defined in lua.h)
- **LUALIB_API**: Macro controlling visibility of library functions (likely defined in luaconf.h, included via lua.h)
