# Source_Files/Lua/lua_serialize.cpp

## File Purpose
Serializes and deserializes Lua objects (tables, numbers, strings, userdata, etc.) to/from binary streams. Supports version-controlled binary format with reference tracking to handle shared and recursive structures. Entry points are `lua_save()` and `lua_restore()` for use by the Lua runtime.

## Core Responsibilities
- Recursively serialize Lua values (nil, number, boolean, string, table, userdata) to binary format
- Track object references to avoid duplicating shared values and handle cyclic references
- Validate table keys to ensure only serializable types are used
- Restore Lua values from binary stream in matching format
- Handle userdata by storing metatable name and index; reconstruct via `__new` metamethod
- Manage version checking during deserialization; reject newer formats
- Log errors on I/O failures during serialization/deserialization

## Key Types / Data Structures
None (all types are from Lua C API or standard library).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SAVED_REFERENCE_PSEUDOTYPE` | `const static int` | file-static | Pseudo-type value (-2) written to stream to mark reference lookups |
| `kVersion` | `const uint16` | file-static | Binary format version (currently 1) for compatibility checks |

## Key Functions / Methods

### valid_key
- Signature: `static bool valid_key(int type)`
- Purpose: Check if a Lua type is valid as a table key during serialization.
- Inputs: Lua type constant (e.g., `LUA_TNUMBER`, `LUA_TSTRING`)
- Outputs/Return: `true` if type is number, boolean, string, table, or userdata; `false` otherwise
- Side effects: None
- Calls: None
- Notes: Silently skips invalid keys (e.g., functions, nil) during table enumeration in `save()`.

### save
- Signature: `static void save(lua_State *L, BOStreamBE& s, uint32& counter)`
- Purpose: Recursively serialize the top Lua stack value to binary stream.
- Inputs: Lua state, output stream, reference counter (passed by reference)
- Outputs/Return: None (writes to stream `s`)
- Side effects: Writes binary data to stream; adds object to reference table on stack[1] for tables/userdata; increments `counter`
- Calls: `lua_pushvalue`, `lua_rawget`, `lua_pushnil`, `lua_next`, `lua_getmetatable`, `lua_gettable`, `lua_getfield` (Lua C API); recursive `save()`
- Notes: 
  - Checks if object is already in reference table (stack[1]); if so, writes a reference pseudotype instead of full serialization
  - Tables and userdata are assigned a reference number for sharing detection
  - Table enumeration writes key-value pairs; terminates with nil key
  - Userdata serializes the metatable name and an `"index"` field from the metatable

### lua_save
- Signature: `bool lua_save(lua_State *L, std::streambuf* sb)`
- Purpose: Public entry point to serialize the top-of-stack Lua value to a stream.
- Inputs: Lua state (with value on top of stack), output streambuf pointer
- Outputs/Return: `true` on success; `false` on stream I/O failure
- Side effects: Asserts that stack depth is exactly 1 at entry; creates and removes a reference table on the stack; writes version number and serialized value to stream
- Calls: `lua_newtable`, `lua_insert`, `save()`, `lua_remove`, `logWarning1`
- Notes: Asserts precondition; catches `basic_bstream::failure` exceptions and logs them; clears Lua stack on exception for safety.

### restore
- Signature: `static int restore(lua_State *L, BIStreamBE& s)`
- Purpose: Recursively deserialize a single Lua value from binary stream and push onto Lua stack.
- Inputs: Lua state, input stream
- Outputs/Return: Type code of deserialized value (`LUA_TNIL`, `LUA_TNUMBER`, etc.)
- Side effects: Pushes deserialized value onto stack; adds tables/userdata to reference table on stack[1]; calls Lua metamethods (e.g., `__new` on userdata)
- Calls: `lua_pushnil`, `lua_pushboolean`, `lua_pushnumber`, `lua_pushlstring`, `lua_newtable`, `lua_rawset`, `lua_gettable`, `lua_getfield`, `lua_call`, `lua_remove` (Lua C API); recursive `restore()`
- Notes:
  - Returns the type code so callers know what was pushed
  - Table deserialization loops on key-value pairs until a nil key is encountered
  - Userdata calls `__new` metamethod from registry to reconstruct object
  - Reference pseudotype performs a lookup in reference table (stack[1])

### lua_restore
- Signature: `bool lua_restore(lua_State *L, std::streambuf* sb)`
- Purpose: Public entry point to deserialize a Lua value from a stream and place it on the stack.
- Inputs: Lua state, input streambuf pointer
- Outputs/Return: `true` on success; `false` on version mismatch or stream I/O failure
- Side effects: Creates and removes a reference table on the stack; pushes deserialized value onto stack; clears stack on exception
- Calls: `lua_newtable`, `restore()`, `lua_remove`, `logWarning`, `logWarning1`
- Notes: Checks version compatibility; rejects data if saved version > current `kVersion`; catches stream failures and logs them.

## Control Flow Notes
- **Save flow**: `lua_save()` initializes version and reference table, then delegates to `save()` for recursive serialization of the root object.
- **Restore flow**: `lua_restore()` checks version, initializes reference table, then delegates to `restore()` for recursive deserialization.
- **Table handling**: Tables and userdata are tracked by reference number to enable sharing. The reference table (stack[1]) maps object identity ΓåÆ reference number for saving, and reference number ΓåÆ object for loading.
- **Lua stack usage**: Both functions expect one value on input and maintain a reference table at stack[1] during processing; it is removed before return.

## External Dependencies
- **Lua C API**: `lua.h`, `lauxlib.h`, `lualib.h` (lua_State, lua_pushvalue, lua_rawget, lua_type, lua_tonumber, lua_tostring, etc.)
- **Serialization**: `BStream.h` ΓÇö `BOStreamBE` (output stream), `BIStreamBE` (input stream), `basic_bstream::failure` exception
- **Logging**: `Logging.h` ΓÇö `logWarning()`, `logWarning1()` for error reporting
- **Standard library**: `<vector>` for buffering; `<streambuf>` for stream abstraction
- **Config**: `config.h` ΓÇö conditional compilation guard `HAVE_LUA`
