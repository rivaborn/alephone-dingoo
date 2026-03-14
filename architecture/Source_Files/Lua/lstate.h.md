# Source_Files/Lua/lstate.h

## File Purpose
Core Lua 5.1 VM state header defining the runtime representation of global and per-thread execution states. Manages call stack metadata, garbage collection state, string interning, memory allocation, and thread management for the embedded Lua runtime in Aleph One.

## Core Responsibilities
- Define `global_State` struct: shared state across all Lua threads (memory allocator, GC lists, string table, metatables, registry)
- Define `lua_State` struct: per-thread execution state (stack, call info, error handling, hooks, environment)
- Define `stringtable` for string interning with collision chains
- Define `CallInfo` for call stack frame metadata
- Define `GCObject` union: all garbage-collectable types (strings, tables, functions, threads, etc.)
- Provide access macros (`gt()`, `G()`, `registry()`) and type-checking/conversion macros
- Declare thread creation/destruction entry points

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `stringtable` | struct | Hash table for string interning; tracks `nuse` count and `size`; prevents duplicate string allocations |
| `CallInfo` | struct | Call frame metadata: `base` (stack frame origin), `func` (function value ptr), `top` (stack top for this call), `savedpc` (return address), `nresults` (expected return count), `tailcalls` (tail call tracking) |
| `global_State` | struct | Shared VM state across all threads: string table, GC allocator/state, registry, metatables, tag method names, error panic handler |
| `lua_State` | struct | Per-thread execution state: evaluation stack, call stack (`base_ci` array), current CallInfo, hook state, error recovery jumppoint, open upvalues list |
| `GCObject` | union | All collectable types: `TString`, `Udata`, `Closure`, `Table`, `Proto`, `UpVal`, `lua_State` (threads) |

## Global / File-Static State
None.

## Key Functions / Methods

### luaE_newthread
- Signature: `lua_State *luaE_newthread (lua_State *L);`
- Purpose: Create a new Lua thread (coroutine) sharing the given state's global state
- Inputs: Pointer to an existing lua_State to access global state
- Outputs/Return: New lua_State pointer; NULL if allocation fails
- Side effects: Allocates thread struct, stack, call info array; registers in global state
- Calls: (declared; defined in ldo.c)
- Notes: New thread shares `global_State` but has independent execution stack and call stack

### luaE_freethread
- Signature: `void luaE_freethread (lua_State *L, lua_State *L1);`
- Purpose: Deallocate a Lua thread and its resources
- Inputs: L (main/access thread for global state), L1 (thread to free)
- Outputs/Return: void
- Side effects: Deallocates L1's stack, call info array, error recovery state; removes from global lists
- Calls: (declared; defined in ldo.c)
- Notes: Must not be called on the main thread

## Control Flow Notes
**Thread lifecycle**: New threads allocated via `luaE_newthread`, execute independently via interpreter loop, freed via `luaE_freethread`.

**Call stack**: Each `lua_State` maintains `base_ci` array of `CallInfo` records; `ci` points to current frame. On function call, a new `CallInfo` is pushed; on return, popped.

**GC integration**: `global_State` tracks GC lists (rootgc, gray, weak, etc.); sweep state managed across all threads via shared state.

## External Dependencies
- `lua.h` ΓÇô Lua public API types and constants (`lua_State`, `LUA_*` type tags)
- `lobject.h` ΓÇô Lua value types (`TValue`, `GCObject`, `TString`, `Table`, `Closure`, `Proto`, `UpVal`)
- `ltm.h` ΓÇô Tag method enumeration and metatable lookups
- `lzio.h` ΓÇô Buffered I/O structures (`Mbuffer` for chunk loading)
- `ldo.c` ΓÇô Implementation of `luaE_newthread`, `luaE_freethread`, and `lua_longjmp` definition
- Type references: `Instruction` (VM opcode), `lua_Alloc` (memory allocator), `lua_Hook` (debug hook), `lua_CFunction` (C function callback)
