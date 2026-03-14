# Source_Files/Lua/lobject.h

## File Purpose
Defines internal memory representation and type system for Lua 5.1 runtime objects. Declares tagged value structures (TValue), garbage-collected types, and accessor/setter macros for the virtual machine to manipulate objects uniformly.

## Core Responsibilities
- Define type tags and distinguish collectable vs non-collectable values
- Provide TValue (tagged value) union structure combining value data + type tag
- Define GCObject union encapsulating all garbage-collected types
- Declare String (TString), Userdata (Udata), Function Prototype (Proto), Closure, and Table structures
- Provide type-checking macros (ttisnil, ttisnumber, etc.)
- Provide value accessor macros (ttype, gcvalue, nvalue, clvalue, hvalue, etc.)
- Provide value setter macros (setnilvalue, setnvalue, setsvalue, etc.) with liveness checks
- Define table hashing and size computation utilities

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| GCObject | union | Discriminated union holding any garbage-collected object (string, userdata, function, table, proto, upval, thread) |
| Value | union | Untagged value data: GC pointer, void pointer, number, or boolean |
| TValue | struct | Tagged value: {Value, int tt}; the fundamental runtime value type |
| TString | union | String header with alignment dummy, metadata (reserved, hash, len) |
| Udata | union | Userdata header with metatable, environment, length |
| Proto | struct | Function prototype: bytecode, constants, nested protos, debug info (lineinfo, locvars, upvalue names) |
| UpVal | struct | Upvalue: pointer to stack/closed value, union holding closed value or doubly-linked list for open state |
| CClosure | struct | C function closure: C function pointer, 1+ upvalue slots |
| LClosure | struct | Lua function closure: Proto pointer, upvalue array |
| Closure | union | Discriminated closure: CClosure or LClosure |
| Table | struct | Hash table: array part, hash part (node array), metatable, size metadata |
| Node | struct | Hash table entry: {TValue i_val, TKey i_key} |
| TKey | union | Table key: tagged value + next pointer (for chaining), or raw TValue |
| LocVar | struct | Local variable debug metadata: name, startpc, endpc |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| luaO_nilobject_ | TValue | global | Sentinel nil value referenced by luaO_nilobject macro |

## Key Functions / Methods

### luaO_log2
- Signature: `int luaO_log2(unsigned int x)`
- Purpose: Compute floor(logΓéé(x))
- Inputs: unsigned int x
- Outputs/Return: logΓéé(x) as int
- Side effects: none
- Calls: defined elsewhere
- Notes: used in ceillog2 macro for table sizing

### luaO_int2fb
- Signature: `int luaO_int2fb(unsigned int x)`
- Purpose: Convert integer to float-bits representation (compression)
- Inputs: unsigned int x
- Outputs/Return: compressed representation as int
- Side effects: none
- Calls: defined elsewhere

### luaO_fb2int
- Signature: `int luaO_fb2int(int x)`
- Purpose: Decompress float-bits back to integer
- Inputs: int x (compressed float bits)
- Outputs/Return: decompressed int
- Side effects: none
- Calls: defined elsewhere

### luaO_rawequalObj
- Signature: `int luaO_rawequalObj(const TValue *t1, const TValue *t2)`
- Purpose: Raw equality test for two TValues (no metamethods)
- Inputs: two TValue pointers
- Outputs/Return: 1 if equal, 0 otherwise
- Side effects: none
- Calls: defined elsewhere

### luaO_str2d
- Signature: `int luaO_str2d(const char *s, lua_Number *result)`
- Purpose: Parse string to double/number, store in result
- Inputs: string, result pointer
- Outputs/Return: success status
- Side effects: writes to *result
- Calls: defined elsewhere

### luaO_pushvfstring
- Signature: `const char *luaO_pushvfstring(lua_State *L, const char *fmt, va_list argp)`
- Purpose: Printf-style string formatting, push result onto Lua stack
- Inputs: Lua state, format string, va_list of arguments
- Outputs/Return: pointer to formatted string
- Side effects: pushes string onto stack
- Calls: defined elsewhere

### luaO_pushfstring
- Signature: `const char *luaO_pushfstring(lua_State *L, const char *fmt, ...)`
- Purpose: Variadic wrapper around luaO_pushvfstring
- Inputs: Lua state, format string, variadic arguments
- Outputs/Return: pointer to formatted string
- Side effects: pushes string onto stack
- Calls: luaO_pushvfstring

### luaO_chunkid
- Signature: `void luaO_chunkid(char *out, const char *source, size_t len)`
- Purpose: Generate human-readable chunk identifier from source name
- Inputs: output buffer, source string, length
- Outputs/Return: none
- Side effects: writes to *out
- Calls: defined elsewhere

## Control Flow Notes
Header-only definitions; no control flow. Provides building blocks for VM instruction execution (frame setup/teardown, type dispatch, value access). Used pervasively in lvm.c, lapi.c, ltable.c, and other core modules to represent and manipulate runtime state.

## External Dependencies
- **config.h**: HAVE_LUA conditional guard
- **llimits.h**: lu_byte, lu_int32, lu_mem, l_mem, L_Umaxalign, Instruction, cast, check_exp, lua_assert macros
- **lua.h**: type tag constants (LUA_TNIL, LUA_TNUMBER, etc.), lua_Number, lua_State, lua_CFunction types, LUA_IDSIZE constant
- **stdarg.h**: va_list for variadic functions
