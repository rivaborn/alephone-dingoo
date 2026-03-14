# Source_Files/Lua/lua_serialize.h

## File Purpose
Declares a serialization interface for Lua objects. Provides functions to save Lua values from the interpreter stack to a C++ stream buffer and restore them back, enabling persistence of game state or configuration data. Bridges Lua's C API with C++ stream utilities.

## Core Responsibilities
- Declare `lua_save()` to serialize Lua stack objects to streams
- Declare `lua_restore()` to deserialize and reconstruct Lua objects
- Provide conditional compilation for Lua support (HAVE_LUA guard)
- Expose a C++ standard library interface (std::streambuf) for Lua serialization

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### lua_save
- Signature: `bool lua_save(lua_State *L, std::streambuf* sb);`
- Purpose: Serialize the Lua object at the top of the stack to a stream buffer
- Inputs: `lua_State *L` (interpreter state), `std::streambuf* sb` (output stream)
- Outputs/Return: `bool` (success/failure)
- Side effects: Reads from Lua stack; writes to stream buffer
- Calls: (defined elsewhere)
- Notes: Expects object on top of Lua stack; implementation handles type detection and encoding

### lua_restore
- Signature: `bool lua_restore(lua_State *L, std::streambuf* sb);`
- Purpose: Deserialize a Lua object from a stream buffer and push it onto the stack
- Inputs: `lua_State *L` (interpreter state), `std::streambuf* sb` (input stream)
- Outputs/Return: `bool` (success/failure); restored object pushed to stack on success
- Side effects: Reads from stream buffer; modifies Lua stack
- Calls: (defined elsewhere)
- Notes: Result pushed to top of stack; implementation handles type reconstruction

## Control Flow Notes
This is a pure declaration file. No control flow inferable. Implementation likely follows an init/save/load pattern where `lua_save()` is called during serialization phases and `lua_restore()` during deserialization phases.

## External Dependencies
- `config.h` ΓÇö Feature detection (HAVE_LUA)
- `cseries.h` ΓÇö Engine series library
- `<streambuf>` ΓÇö C++ standard stream buffering
- Lua C API (wrapped in `extern "C"`): `lua.h`, `lauxlib.h`, `lualib.h`
- Function implementations defined elsewhere (not in this header)
