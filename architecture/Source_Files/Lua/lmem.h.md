# Source_Files/Lua/lmem.h

## File Purpose
Header file providing Lua's memory management interface. Defines macros for safe allocation, reallocation, and deallocation of memory blocks and arrays, with overflow checking and error handling.

## Core Responsibilities
- Declare core memory allocation functions (`luaM_realloc_`, `luaM_toobig`, `luaM_growaux_`)
- Provide convenience macros for single-object allocation (`luaM_new`, `luaM_malloc`)
- Provide vector/array allocation macros (`luaM_newvector`, `luaM_reallocvector`, `luaM_growvector`)
- Implement overflow-safe reallocation with size checking (`luaM_reallocv`)
- Supply standardized error message constant

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MEMERRMSG` | `const char*` (string literal) | static | Standard error message for allocation failures |
| `MAX_SIZET` | `size_t` | static (from llimits.h) | Maximum safe allocation size; prevents overflow |

## Key Functions / Methods

### luaM_realloc_
- **Signature:** `void *luaM_realloc_ (lua_State *L, void *block, size_t oldsize, size_t size);`
- **Purpose:** Core reallocation function; handles allocation, reallocation, and freeing based on size parameter
- **Inputs:** Lua state `L`, existing block pointer, old size, new size
- **Outputs/Return:** Pointer to newly allocated/reallocated block; NULL or error on failure
- **Side effects:** Heap allocation/deallocation; may trigger Lua error on out-of-memory
- **Calls:** (not visible in this file)
- **Notes:** Behavior depends on parameters: `oldsize=0, size>0` = allocate; `size=0` = free; `size>0, oldsize>0` = realloc

### luaM_toobig
- **Signature:** `void *luaM_toobig (lua_State *L);`
- **Purpose:** Error handler called when requested allocation exceeds `MAX_SIZET`
- **Inputs:** Lua state `L`
- **Outputs/Return:** Never returns (noreturn)
- **Side effects:** Raises Lua error with memory message
- **Notes:** Prevents integer overflow in size calculations

### luaM_growaux_
- **Signature:** `void *luaM_growaux_ (lua_State *L, void *block, int *size, size_t size_elem, int limit, const char *errormsg);`
- **Purpose:** Helper for dynamically growing vectors; manages geometric expansion strategy
- **Inputs:** Lua state, existing block, pointer to current size (in/out), element size, size limit, error message
- **Outputs/Return:** Pointer to reallocated block; updates `*size` in place
- **Side effects:** Reallocates memory; raises error if limit exceeded
- **Notes:** Used by `luaM_growvector` macro; size parameter is updated during growth

---

**Key Macros** (summaries):
- `luaM_reallocv`: Safe vector reallocation with overflow check; calls `luaM_toobig` if `(n)+1 > MAX_SIZET/(e)`
- `luaM_malloc`, `luaM_new`: Allocation from NULL pointer
- `luaM_free`, `luaM_freemem`, `luaM_freearray`: Deallocation wrappers
- `luaM_growvector`: Conditional growth; reallocates if needed capacity exceeded

## Control Flow Notes
Memory macros are called during all phases:
- **Init:** Creating runtime structures (stacks, tables, strings)
- **Frame/Update:** Expanding vectors during table insertions, string concatenation
- **Shutdown:** Releasing all allocations (in reverse order)

The `+1` guard in `luaM_reallocv` prevents false negatives on size boundaries.

## External Dependencies
- `config.h`: Conditional `HAVE_LUA` guard
- `llimits.h`: `MAX_SIZET`, `cast()` macro
- `lua.h`: `lua_State` opaque type
- Implementation file: `lmem.c` (not shown)
