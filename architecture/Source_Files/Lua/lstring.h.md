# Source_Files/Lua/lstring.h

## File Purpose
Header file for Lua's string table (interned string pool). Defines the public interface for creating and managing strings and userdata, including sizing macros and string creation convenience wrappers. All strings created via Lua are stored in a hash table managed by this module.

## Core Responsibilities
- Define macros to calculate memory size of strings (`TString`) and userdata (`Udata`)
- Provide convenience macros for string creation from C strings and string literals
- Declare interface for resizing the string table and allocating new userdata
- Mark strings as "fixed" (exempt from garbage collection)

## Key Types / Data Structures
None (types are defined in included headers: `TString` and `Udata` from `lobject.h`).

## Global / File-Static State
None.

## Key Functions / Methods

### luaS_resize
- Signature: `void luaS_resize(lua_State *L, int newsize)`
- Purpose: Resize the string table hash table to a new capacity.
- Inputs: Lua state `L`, new size for hash table.
- Outputs/Return: None.
- Side effects: Modifies the string table in global state; likely rehashes all interned strings.
- Calls: Not visible in this file.
- Notes: Called during GC to balance memory usage against lookup performance.

### luaS_newudata
- Signature: `Udata *luaS_newudata(lua_State *L, size_t s, Table *e)`
- Purpose: Allocate a new userdata object with specified size and environment table.
- Inputs: `L` (Lua state), `s` (size of userdata payload), `e` (environment table).
- Outputs/Return: Pointer to newly allocated `Udata`.
- Side effects: Allocates memory, registers with GC.
- Calls: Not visible in this file.
- Notes: Userdata is a container for arbitrary C data with an associated metatable and environment.

### luaS_newlstr
- Signature: `TString *luaS_newlstr(lua_State *L, const char *str, size_t l)`
- Purpose: Create a new interned string of specified length.
- Inputs: `L` (Lua state), `str` (C string buffer), `l` (length in bytes).
- Outputs/Return: Pointer to interned `TString`.
- Side effects: Allocates, hashes, and registers string in global string pool; registered with GC.
- Calls: Not visible in this file.
- Notes: If string already exists in pool, returns existing entry. Implements string interning.

## Macros

| Macro | Purpose |
|-------|---------|
| `sizestring(s)` | Compute total memory size of a `TString` object (header + character buffer). |
| `sizeudata(u)` | Compute total memory size of a `Udata` object (header + payload). |
| `luaS_new(L, s)` | Convenience wrapper: create string from null-terminated C string (calls `luaS_newlstr` with `strlen`). |
| `luaS_newliteral(L, s)` | Convenience wrapper: create string from string literal, computing length at compile time. |
| `luaS_fix(s)` | Mark a string as fixed by setting the `FIXEDBIT` in its `marked` field (prevents GC collection). |

## Control Flow Notes
This header is part of Lua's string interning system. Strings created by Lua scripts or C API calls flow through `luaS_newlstr`, which checks the string table for duplicates. The string table is managed as part of the global garbage collector state (see `lstate.h:global_State.strt`). During GC sweeps, `luaS_resize` may be called to rebalance the hash table.

## External Dependencies
- `config.h` ΓÇô Platform configuration.
- `lgc.h` ΓÇô Garbage collector macros: `l_setbit`, `FIXEDBIT` bit manipulation.
- `lobject.h` ΓÇô Type definitions: `TString`, `Udata`, `GCObject`.
- `lstate.h` ΓÇô Lua state type: `lua_State`.
