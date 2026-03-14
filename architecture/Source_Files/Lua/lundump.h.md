# Source_Files/Lua/lundump.h

## File Purpose
Header file providing the interface for Lua binary chunk serialization and deserialization. Declares functions to load precompiled bytecode from binary streams, create binary file headers, and dump function prototypes to binary format for caching or distribution.

## Core Responsibilities
- Declare loader interface for deserializing Lua bytecode from binary (undump)
- Declare dumper interface for serializing Lua function prototypes to binary (dump)
- Declare binary header generation interface for chunk files
- Declare debug/inspection printer for bytecode introspection (conditional)
- Define constants for Lua binary format versioning and structure sizes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Proto | struct (in lobject.h) | Lua function prototype containing bytecode, constants, nested functions, debug info |
| ZIO | struct (in lzio.h) | Buffered input stream for reading serialized binary data |
| Mbuffer | struct (in lzio.h) | Memory buffer for intermediate serialization/deserialization work |

## Global / File-Static State
None.

## Key Functions / Methods

### luaU_undump
- **Signature:** `Proto* luaU_undump(lua_State* L, ZIO* Z, Mbuffer* buff, const char* name)`
- **Purpose:** Deserialize and load a single precompiled Lua function prototype from a binary stream.
- **Inputs:** Lua state (L), input stream (Z), working buffer (buff), source name for diagnostics (name)
- **Outputs/Return:** Pointer to reconstructed Proto object; NULL on error
- **Side effects:** Allocates memory via lua_State; reads from ZIO; modifies Mbuffer
- **Calls:** (implemented in lundump.c, not visible here)
- **Notes:** Validates binary format; handles version checking internally

### luaU_header
- **Signature:** `void luaU_header(char* h)`
- **Purpose:** Generate a binary file header for Lua chunk files.
- **Inputs:** Pointer to buffer where header will be written (h)
- **Outputs/Return:** None; writes LUAC_HEADERSIZE (12) bytes to buffer
- **Side effects:** Writes to provided buffer (user responsible for allocation)
- **Calls:** (implemented in lundump.c)
- **Notes:** Header encodes version (0x51 = Lua 5.1) and format type (0 = official)

### luaU_dump
- **Signature:** `int luaU_dump(lua_State* L, const Proto* f, lua_Writer w, void* data, int strip)`
- **Purpose:** Serialize a Lua function prototype to binary format via callback writer.
- **Inputs:** Lua state (L), prototype to dump (f), writer callback (w), callback context (data), strip debug info flag
- **Outputs/Return:** Integer status (0 = success)
- **Side effects:** Invokes writer callback repeatedly to output binary data
- **Calls:** (implemented in ldump.c)
- **Notes:** `strip` parameter excludes debug info (line numbers, local variable names) if non-zero

### luaU_print
- **Signature:** `void luaU_print(const Proto* f, int full)`
- **Purpose:** Print human-readable representation of a Lua prototype structure.
- **Inputs:** Prototype (f), verbosity flag (full)
- **Outputs/Return:** None; output to stdio
- **Side effects:** Prints to stdout
- **Calls:** (implemented in print.c)
- **Notes:** Only declared when `luac_c` is defined; for bytecode inspection tool (luac compiler)

## Control Flow Notes
Part of Lua's bytecode I/O pipeline:
- **Loading:** undump deserializes cached/distributed bytecode during script execution
- **Compilation:** dump serializes compiled prototypes to binary cache or standalone chunks
- **Debugging:** print introspects bytecode structure (luac compiler utility)

## External Dependencies
- `config.h` ΓÇö `HAVE_LUA` preprocessor guard
- `lobject.h` ΓÇö Proto struct definition, GCObject types
- `lzio.h` ΓÇö ZIO and Mbuffer structures
- `lua.h` ΓÇö lua_State opaque type, lua_Writer callback typedef, LUAI_FUNC macro (defined elsewhere)
