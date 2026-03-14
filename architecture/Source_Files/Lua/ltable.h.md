# Source_Files/Lua/ltable.h

## File Purpose
Header defining the public C interface for Lua's hash table (associative array) implementation. Declares functions for table creation, lookup, assignment, resizing, and iteration. Part of the Lua runtime embedded in the AlephOne game engine.

## Core Responsibilities
- Define accessor macros (`gnode`, `gkey`, `gval`, `gnext`, `key2tval`) for internal table node structures
- Declare typed lookup/assignment functions (by integer key, string key, or generic TValue)
- Declare table lifecycle operations (creation, resizing, freeing)
- Declare table iteration and length queries
- Expose debug introspection functions (conditional on `LUA_DEBUG`)

## Key Types / Data Structures
Defined in included `lobject.h`:
| Name | Kind | Purpose |
|------|------|---------|
| Table | struct | Hash table with array part, hash part (nodes), metadata, and GC linkage |
| Node | struct | Single hash entry: `i_val` (value) and `i_key` (TKey with chaining) |
| TKey | union | Tagged key with embedded `next` pointer for collision chaining |
| TValue | struct | Tagged value (type + union of gc/pointer/number/boolean) |
| TString | union | String object (hash table key type) |

## Global / File-Static State
None.

## Key Functions / Methods

### luaH_get / luaH_set
- **Signature:** `const TValue *luaH_get(Table *t, const TValue *key)` / `TValue *luaH_set(lua_State *L, Table *t, const TValue *key)`
- **Purpose:** Generic key lookup and assignment; dispatches to array or hash part based on key type
- **Inputs:** Table pointer, key as tagged value (TValue)
- **Outputs/Return:** Pointer to value in table (const for get, mutable for set)
- **Side effects:** `luaH_set` may trigger array resizing or hash reorganization; modifies table state
- **Notes:** Core O(1) lookup/insert operations; used by VM for `t[k]` syntax

### luaH_getnum / luaH_setnum
- **Signature:** `const TValue *luaH_getnum(Table *t, int key)` / `TValue *luaH_setnum(lua_State *L, Table *t, int key)`
- **Purpose:** Fast path for integer-key lookups/assignments; bypasses type checking
- **Inputs:** Table, integer index
- **Outputs/Return:** Pointer to value
- **Side effects:** `luaH_setnum` may trigger resizing
- **Notes:** Common case optimization; integers stored in array part when in bounds

### luaH_getstr / luaH_setstr
- **Signature:** `const TValue *luaH_getstr(Table *t, TString *key)` / `TValue *luaH_setstr(lua_State *L, Table *t, TString *key)`
- **Purpose:** Fast path for string keys (interned TString pointers)
- **Inputs:** Table, TString pointer
- **Outputs/Return:** Pointer to value
- **Side effects:** `luaH_setstr` may trigger hash reorganization
- **Notes:** String keys use pointer equality; hash collision resolution via `next` chaining

### luaH_new
- **Signature:** `Table *luaH_new(lua_State *L, int narray, int lnhash)`
- **Purpose:** Create a new table with preallocated array and hash parts
- **Inputs:** Lua state, array size, logΓéé of hash size
- **Outputs/Return:** Pointer to newly allocated Table
- **Side effects:** Allocates memory; marks as GC object
- **Notes:** `lnhash` is logΓéé, so actual hash size is 2^lnhash

### luaH_next
- **Signature:** `int luaH_next(lua_State *L, Table *t, StkId key)`
- **Purpose:** Iterate over table entries (for-in loop semantics)
- **Inputs:** Lua state, table, key pointer (modified in-place to next key)
- **Outputs/Return:** Non-zero if more entries, zero at end
- **Side effects:** Writes next key to stack location
- **Notes:** Stateful iteration; caller must manage key state

### Helper functions
- `luaH_resizearray`: Resize array part to new size (memory allocation/reorganization)
- `luaH_free`: Free table and associated memory
- `luaH_getn`: Return effective table length (array size + hash entries)
- `luaH_mainposition` (debug): Return canonical hash slot for a key
- `luaH_isdummy` (debug): Test if node is sentinel

## Control Flow Notes
Accessed during:
- **Initialization:** `luaH_new` called when scripts create tables
- **Frame/VM execution:** `luaH_get`, `luaH_set`, typed variants called continuously for `table[key]` operations
- **Iteration:** `luaH_next` drives for-in loops
- **Cleanup/GC:** `luaH_free` called during garbage collection phase

## External Dependencies
- `config.h` ΓÇö Provides `HAVE_LUA` guard; configuration flags (AlephOne-specific)
- `lobject.h` ΓÇö Defines `Table`, `Node`, `TValue`, `TKey`, `TString`, `StkId` types and TValue/GC macros
- `LUAI_FUNC` macro ΓÇö Controls function visibility/export (likely `extern` or `__declspec`)
- Lua state type `lua_State` ΓÇö Defined elsewhere (memory allocator context)
