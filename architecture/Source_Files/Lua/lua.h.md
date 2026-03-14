# Source_Files/Lua/lua.h

## File Purpose

Public C API header for Lua 5.1.2, an extensible extension language. Defines the interface for embedding Lua in C/C++ applications, including state management, stack operations, type system, and execution control.

## Core Responsibilities

- **State lifecycle**: Create, manage, and destroy Lua VM instances
- **Stack API**: Push/pop values, manipulate stack positions, type introspection
- **Value conversion**: Bridge between C native types and Lua types
- **Execution**: Load and execute Lua code, call functions, handle coroutines
- **Memory management**: Custom allocator support, garbage collection control
- **Debug API**: Stack introspection, hook installation, breakpoint support
- **Type system**: Define 8 core types (nil, boolean, number, string, table, function, userdata, thread)
- **C function callbacks**: Register native C functions callable from Lua

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `lua_State` | struct (opaque) | VM instance; all API calls receive a pointer to this |
| `lua_Debug` | struct | Activation record for debug info; contains event, name, source, line numbers |
| `lua_CFunction` | function pointer | Signature for C functions callable from Lua: `int (*)(lua_State *L)` |
| `lua_Reader` | function pointer | Chunk reader for `lua_load`: `const char *(*)(lua_State *L, void *ud, size_t *sz)` |
| `lua_Writer` | function pointer | Chunk writer for `lua_dump`: `int (*)(lua_State *L, const void *p, size_t sz, void *ud)` |
| `lua_Alloc` | function pointer | Custom memory allocator: `void *(*)(void *ud, void *ptr, size_t osize, size_t nsize)` |
| `lua_Hook` | function pointer | Debug hook callback: `void (*)(lua_State *L, lua_Debug *ar)` |

## Global / File-Static State

None.

## Key Functions / Methods

### lua_newstate
- **Signature:** `lua_State *(lua_newstate)(lua_Alloc f, void *ud)`
- **Purpose:** Create a new Lua VM instance with custom memory allocator.
- **Inputs:** `f` = allocator function, `ud` = opaque userdata passed to allocator
- **Outputs/Return:** Pointer to new `lua_State`, or NULL on allocation failure
- **Side effects:** Allocates VM memory; initializes global state
- **Calls:** User-provided allocator `f`
- **Notes:** Paired with `lua_close`

### lua_close
- **Signature:** `void (lua_close)(lua_State *L)`
- **Purpose:** Destroy a Lua VM instance and release all resources.
- **Inputs:** `L` = VM to destroy
- **Outputs/Return:** None
- **Side effects:** Frees all memory associated with `L`; calls finalizers
- **Calls:** User-provided allocator
- **Notes:** No-op if `L` is NULL

### lua_newthread
- **Signature:** `lua_State *(lua_newthread)(lua_State *L)`
- **Purpose:** Create a new coroutine (thread) within the same VM.
- **Inputs:** `L` = parent VM
- **Outputs/Return:** Pointer to new coroutine state; also pushes it on parent stack
- **Side effects:** Allocates coroutine, modifies parent stack
- **Calls:** Memory allocator
- **Notes:** Coroutine shares global state with parent

### lua_gettop
- **Signature:** `int (lua_gettop)(lua_State *L)`
- **Purpose:** Get the current stack height (number of values on stack).
- **Inputs:** `L` = VM
- **Outputs/Return:** Stack size (0 if empty)
- **Side effects:** None
- **Notes:** Returns highest valid stack index

### lua_settop
- **Signature:** `void (lua_settop)(lua_State *L, int idx)`
- **Purpose:** Set stack top to absolute position, truncating or padding with nils.
- **Inputs:** `L` = VM, `idx` = new top (negative indices count from current top)
- **Outputs/Return:** None
- **Side effects:** Modifies stack; may allocate if expanding
- **Notes:** `lua_pop(L, n)` is a macro: `lua_settop(L, -(n)-1)`

### lua_pushvalue / lua_pushnil / lua_pushnumber / lua_pushstring / lua_pushboolean
- **Signature:** `void (lua_pushvalue)(lua_State *L, int idx)` / `void (lua_pushnil)(lua_State *L)` / etc.
- **Purpose:** Push a copy of stack value or a new literal onto stack.
- **Inputs:** `idx` (for pushvalue), value/string/number
- **Outputs/Return:** None
- **Side effects:** Increments stack top; may allocate
- **Notes:** `lua_pushfstring` supports printf-style formatting; `lua_pushcclosure` wraps C function with upvalues

### lua_call
- **Signature:** `void (lua_call)(lua_State *L, int nargs, int nresults)`
- **Purpose:** Call a Lua function (or C closure) already on stack.
- **Inputs:** `nargs` = number of arguments below function, `nresults` = desired return count (or LUA_MULTRET=-1 for all)
- **Outputs/Return:** None (results left on stack)
- **Side effects:** Executes bytecode/C code; may trigger GC, raise errors via longjmp
- **Calls:** Lua functions, C callbacks
- **Notes:** Stack must have function + args; on error, throws via LUAI_THROW

### lua_pcall
- **Signature:** `int (lua_pcall)(lua_State *L, int nargs, int nresults, int errfunc)`
- **Purpose:** Protected call; like `lua_call` but catches errors.
- **Inputs:** Same as `lua_call`, plus `errfunc` = error handler index (0 for none)
- **Outputs/Return:** 0 on success; LUA_ERR* code on failure (error value on stack)
- **Side effects:** Same as `lua_call`, but does not propagate errors
- **Notes:** Preferred for embedding; safe error handling

### lua_load
- **Signature:** `int (lua_load)(lua_State *L, lua_Reader reader, void *dt, const char *chunkname)`
- **Purpose:** Load Lua bytecode or source code into memory (does not execute).
- **Inputs:** `reader` = chunk-reading callback, `dt` = opaque userdata for reader, `chunkname` = debug name
- **Outputs/Return:** 0 on success; LUA_ERR* on error (result/error on stack)
- **Side effects:** Allocates function object; modifies stack
- **Notes:** Recognizes precompiled code by magic `\033Lua` signature

### lua_yield
- **Signature:** `int (lua_yield)(lua_State *L, int nresults)`
- **Purpose:** Suspend a coroutine and return control to caller.
- **Inputs:** `nresults` = number of return values
- **Outputs/Return:** Returns to caller (not a normal function return)
- **Side effects:** Suspends coroutine state
- **Notes:** Only valid inside coroutines (not main thread)

### lua_resume
- **Signature:** `int (lua_resume)(lua_State *L, int narg)`
- **Purpose:** Resume a suspended coroutine.
- **Inputs:** `L` = coroutine, `narg` = number of arguments
- **Outputs/Return:** LUA_YIELD if suspended, 0 if finished, error code on failure
- **Side effects:** Resumes execution; modifies stack
- **Notes:** Arguments left on stack after call

### lua_gc
- **Signature:** `int (lua_gc)(lua_State *L, int what, int data)`
- **Purpose:** Control garbage collector.
- **Inputs:** `what` = LUA_GC* action (STOP, RESTART, COLLECT, COUNT, etc.), `data` = action parameter
- **Outputs/Return:** Depends on action (e.g., memory count for GCCOUNT)
- **Side effects:** May trigger collection; modifies GC state
- **Notes:** LUA_GCCOUNT/GCCOUNTB report memory in KB/bytes

### lua_next
- **Signature:** `int (lua_next)(lua_State *L, int idx)`
- **Purpose:** Iterate table at `idx`; pops key, pushes next (key, value) pair.
- **Inputs:** `idx` = table position; stack must have key (or nil for first)
- **Outputs/Return:** 0 if no more entries; 1 if key/value pushed
- **Side effects:** Modifies stack
- **Notes:** Standard Lua iteration pattern: initialize with nil, loop while `lua_next` returns 1

### lua_error
- **Signature:** `int (lua_error)(lua_State *L)`
- **Purpose:** Raise an error using value on top of stack as message.
- **Inputs:** Stack top = error value
- **Outputs/Return:** Does not return (throws)
- **Side effects:** Triggers error propagation
- **Notes:** Returns int for C compatibility (never actually returns)

## Control Flow Notes

This header defines the API layer for a complete scripting VM. Usage pattern:

1. **Initialization:** `lua_newstate()` ΓåÆ creates VM
2. **Load code:** `lua_load()` ΓåÆ loads bytecode/source, leaves function on stack
3. **Execution:** `lua_call()` or `lua_pcall()` ΓåÆ runs code
4. **Stack exchange:** `lua_push*` (CΓåÆLua), `lua_to*` (LuaΓåÆC) for value marshaling
5. **Coroutines:** `lua_newthread()` ΓåÆ create, `lua_resume()`/`lua_yield()` ΓåÆ control
6. **Cleanup:** `lua_close()` ΓåÆ destroy VM

Debug support is optional: install hooks with `lua_sethook()`, query stack with `lua_getstack()` and `lua_getinfo()`.

## External Dependencies

- **config.h**: Conditional compilation (e.g., `HAVE_LUA`)
- **luaconf.h**: Platform-specific configuration (type sizes, allocators, dynamic linking)
- **Standard C**: `<stdarg.h>`, `<stddef.h>` (variadic args, size_t)
- **Internal symbols** (defined elsewhere): All function implementations, `lua_State` struct definition, `LUA_API` macro (usually `extern` or DLL visibility)

---

**Notes:**
- All exported functions are marked `LUA_API` (defined in luaconf.h; typically `extern` or `__declspec(dllexport/dllimport)`)
- Stack indices: positive = absolute (1-based), negative = relative to top (ΓêÆ1 = top)
- Pseudo-indices (< 0) access VM globals/registry without consuming stack: `LUA_REGISTRYINDEX`, `LUA_GLOBALSINDEX`, etc.
- Macro helpers like `lua_pop()`, `lua_newtable()`, `lua_register()` simplify common patterns
