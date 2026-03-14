# Source_Files/Lua/lfunc.h

## File Purpose
Header file declaring auxiliary functions for manipulating Lua function prototypes and closures. Provides memory allocation, upvalue management, and cleanup routines for Lua's closure system.

## Core Responsibilities
- Factory functions for creating Lua and C closures
- Prototype (function template) creation and destruction
- Upvalue (captured variable) creation, lookup, and lifecycle management
- Memory deallocation for prototypes, closures, and upvalues
- Local variable name lookup by program counter

## Key Types / Data Structures
None. All types (`Proto`, `Closure`, `UpVal`, `TValue`, `Table`, `StkId`) are defined in `lobject.h`.

## Global / File-Static State
None.

## Key Functions / Methods

### luaF_newproto
- Signature: `Proto *luaF_newproto (lua_State *L)`
- Purpose: Allocate and initialize a new function prototype
- Inputs: Lua state
- Outputs/Return: Pointer to new `Proto` object
- Side effects: Memory allocation via Lua allocator; GC registration
- Calls: Not visible in header

### luaF_newCclosure
- Signature: `Closure *luaF_newCclosure (lua_State *L, int nelems, Table *e)`
- Purpose: Create a C closure (wrapper for a C function with captured upvalues)
- Inputs: Lua state, number of upvalues, environment table
- Outputs/Return: Pointer to new `Closure` object
- Side effects: Memory allocation; links closure into GC list
- Calls: Not visible in header
- Notes: Size computed via `sizeCclosure(n)` macro

### luaF_newLclosure
- Signature: `Closure *luaF_newLclosure (lua_State *L, int nelems, Table *e)`
- Purpose: Create a Lua closure (compiled Lua function with captured upvalues)
- Inputs: Lua state, number of upvalues, environment table
- Outputs/Return: Pointer to new `Closure` object
- Side effects: Memory allocation; GC registration
- Calls: Not visible in header
- Notes: Size computed via `sizeLclosure(n)` macro

### luaF_newupval
- Signature: `UpVal *luaF_newupval (lua_State *L)`
- Purpose: Allocate a new upvalue (captured variable container)
- Inputs: Lua state
- Outputs/Return: Pointer to new `UpVal` object
- Side effects: Memory allocation
- Calls: Not visible in header

### luaF_findupval
- Signature: `UpVal *luaF_findupval (lua_State *L, StkId level)`
- Purpose: Find or create an upvalue for a variable at a given stack level
- Inputs: Lua state, stack index of variable
- Outputs/Return: Pointer to `UpVal` (found or newly created)
- Side effects: May allocate new upvalues; manages doubly-linked upvalue list
- Calls: Not visible in header
- Notes: Core to closure variable capture

### luaF_close
- Signature: `void luaF_close (lua_State *L, StkId level)`
- Purpose: Close all open upvalues at or above a given stack level
- Inputs: Lua state, stack threshold
- Outputs/Return: None
- Side effects: Converts open upvalues to closed (captured by value)
- Calls: Not visible in header
- Notes: Called during function return or scope exit

### luaF_freeproto, luaF_freeclosure, luaF_freeupval
- Deallocator functions (signatures not expanded; GC callbacks)
- Purpose: Release memory for prototypes, closures, and upvalues
- Side effects: GC-initiated deallocation
- Notes: Triggered by garbage collector, not user code

### luaF_getlocalname
- Signature: `const char *luaF_getlocalname (const Proto *func, int local_number, int pc)`
- Purpose: Retrieve the name of a local variable by index and program counter (for debugging)
- Inputs: Function prototype, local variable index, instruction pointer
- Outputs/Return: String name of variable, or NULL if not found
- Side effects: None
- Calls: Not visible in header
- Notes: Used by debugger and error reporting

## Control Flow Notes
This is a declarative header. Functions are entry points for the Lua runtime's closure subsystem:
- **Initialization**: `luaF_newproto`, `luaF_newCclosure`, `luaF_newLclosure` called during function definition/compilation
- **Execution**: `luaF_findupval` and `luaF_newupval` called during closure instantiation
- **Cleanup**: `luaF_close` called on function exit; `luaF_free*` called by garbage collector

## External Dependencies
- `config.h` ΓÇô Build configuration (e.g., `HAVE_LUA` guard)
- `lobject.h` ΓÇô Type definitions: `Proto`, `Closure`, `UpVal`, `Table`, `TValue`, `StkId`, `lua_State`
- Lua C API: `lua_CFunction` (C function pointer type)
