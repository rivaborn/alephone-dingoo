# Source_Files/Lua/ldo.h

## File Purpose
Header for Lua's stack and call management system. Defines the interface for function call execution, protected execution contexts, and stack/CallInfo array allocation. Central to the VM's call semantics.

## Core Responsibilities
- Stack safety macros (`luaD_checkstack`, `incr_top`) for bounds checking and growth
- Stack pointer serialization/restoration (`savestack`/`restorestack`) for snapshots across reallocation
- CallInfo array index serialization (`saveci`/`restoreci`) for position tracking
- Function call protocol (precall setup, call execution, postcall cleanup)
- Protected execution (`luaD_pcall`, `luaD_rawrunprotected`) for exception handling
- Stack and CallInfo array reallocation and growth
- Parser invocation with error recovery
- Hook invocation for debugger/profiler integration
- Error propagation (`luaD_throw`, `luaD_seterrorobj`)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Pfunc` | typedef | Function pointer type for protected functions; signature `void (*)(lua_State *L, void *ud)` |

## Global / File-Static State
None.

## Key Functions / Methods

### luaD_precall
- Signature: `int luaD_precall(lua_State *L, StkId func, int nresults)`
- Purpose: Prepare a function call (Lua or C) by setting up its CallInfo frame and handling Lua-to-C boundaries.
- Inputs: `L` (VM state), `func` (stack position of function to call), `nresults` (expected return count)
- Outputs/Return: `PCRLUA` (Lua call initiated), `PCRC` (C function called directly), `PCRYIELD` (C function yielded)
- Side effects: Modifies VM call stack and base/top pointers; may allocate CallInfo on demand.
- Calls: Indirectly references macros `luaD_checkstack`, `incr_top`.
- Notes: Returns immediately for C calls; Lua calls require subsequent `luaD_call`. Yield support for continuations.

### luaD_call
- Signature: `void luaD_call(lua_State *L, StkId func, int nResults)`
- Purpose: Execute a Lua function call synchronously, including re-entry from C.
- Inputs: `L` (VM state), `func` (stack position of function), `nResults` (expected return count)
- Outputs/Return: void
- Side effects: Executes bytecode; modifies stack; may trigger GC or hook invocation.
- Calls: Error handling and hook machinery.

### luaD_pcall
- Signature: `int luaD_pcall(lua_State *L, Pfunc func, void *u, ptrdiff_t oldtop, ptrdiff_t ef)`
- Purpose: Run a protected function with automatic exception recovery and error function invocation.
- Inputs: `func` (protected function to run), `u` (opaque user data), `oldtop` (saved stack position), `ef` (error function index)
- Outputs/Return: Error code (0 on success, `LUA_ERR*` on failure).
- Side effects: May restore stack to `oldtop` on error; calls error handler.

### luaD_poscall
- Signature: `int luaD_poscall(lua_State *L, StkId firstResult)`
- Purpose: Finalize a function return by popping its CallInfo frame and repositioning the stack.
- Inputs: `firstResult` (position of first return value)
- Outputs/Return: Success indicator.
- Side effects: Modifies CI and stack state.

### luaD_growstack
- Signature: `void luaD_growstack(lua_State *L, int n)`
- Purpose: Enlarge the stack when `luaD_checkstack` detects insufficient space.
- Inputs: `n` (minimum space needed in TValue units)
- Side effects: Reallocates stack buffer; invalidates all saved stack positions without corresponding restorestack calls.

### luaD_reallocstack
- Signature: `void luaD_reallocstack(lua_State *L, int newsize)`
- Purpose: Resize the stack to a specific capacity (internal, used by growth).
- Inputs: `newsize` (new stack size)
- Side effects: Reallocates and updates all pointers; invalidates saved positions.

### luaD_reallocCI
- Signature: `void luaD_reallocCI(lua_State *L, int newsize)`
- Purpose: Resize the CallInfo array.
- Inputs: `newsize` (new CallInfo count)
- Side effects: Reallocates; invalidates CallInfo pointer snapshots without corresponding restoreci.

### luaD_throw
- Signature: `void luaD_throw(lua_State *L, int errcode)`
- Purpose: Non-local jump (longjmp) to the nearest protection boundary.
- Inputs: `errcode` (error type to propagate)
- Side effects: Never returns; transfers control to error handler.
- Notes: Called when an unrecoverable error occurs in unprotected code.

### luaD_rawrunprotected
- Signature: `int luaD_rawrunprotected(lua_State *L, Pfunc f, void *ud)`
- Purpose: Low-level protected execution without error handler invocation.
- Inputs: `f` (function to run), `ud` (opaque data)
- Outputs/Return: Error code.
- Side effects: Sets up exception handler; restores on error without calling Lua error handler.

### luaD_seterrorobj
- Signature: `void luaD_seterrorobj(lua_State *L, int errcode, StkId oldtop)`
- Purpose: Construct the error object (message or value) on the stack after a protected call fails.
- Inputs: `errcode` (error type), `oldtop` (saved stack position to restore)
- Side effects: Pushes error value onto stack.

### luaD_callhook
- Signature: `void luaD_callhook(lua_State *L, int event, int line)`
- Purpose: Invoke the debugger/profiler hook for call/return/line events.
- Inputs: `event` (hook event type), `line` (source line number)
- Side effects: Calls user-defined hook if enabled.

### luaD_protectedparser
- Signature: `int luaD_protectedparser(lua_State *L, ZIO *z, const char *name)`
- Purpose: Parse Lua source code with error recovery.
- Inputs: `z` (input stream), `name` (chunk name for diagnostics)
- Outputs/Return: Error code.
- Side effects: On success, leaves compiled prototype on stack; on error, leaves error message.

## Control Flow Notes
This file defines the boundaries of **protected execution** (try/catch equivalents) and **function call setup/teardown**:
- **Parse phase**: `luaD_protectedparser` wraps the lexer/parser with exception handling.
- **Precall phase**: `luaD_precall` determines call type (Lua vs. C) and allocates CallInfo.
- **Call phase**: `luaD_call` (Lua) or direct C function invocation; interruptible by hooks and errors.
- **Postcall phase**: `luaD_poscall` cleans up and repositions the stack.
- **Error recovery**: Errors throw (via `luaD_throw`) to the nearest `luaD_pcall` or `luaD_rawrunprotected` boundary.

Stack macros (`checkstack`, `incr_top`, `save/restore`) are used throughout frame setup and bytecode execution to prevent buffer overruns during allocation-heavy operations.

## External Dependencies
- **Includes**: `lobject.h` (TValue, Closure, Proto), `lstate.h` (lua_State, CallInfo, global_State), `lzio.h` (ZIO, input streams)
- **Defined elsewhere**: `lua_State` structure, `TValue` union, `StkId` (alias for `TValue *`), `CallInfo` struct, `Instruction` (bytecode), `lua_Reader` callback
