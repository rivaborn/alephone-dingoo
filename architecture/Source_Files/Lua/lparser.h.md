# Source_Files/Lua/lparser.h

## File Purpose
Header file for Lua's parser and code generator. Defines expression descriptors, function compilation state, and upvalue tracking structures used during parsing and bytecode generation.

## Core Responsibilities
- Define expression kind enum (`expkind`) categorizing all Lua expression types
- Define expression descriptor (`expdesc`) with value encoding and control-flow patch lists
- Define upvalue metadata structure for closure variable handling
- Define function state (`FuncState`) tracking compiler context during code generation
- Declare parser entry point `luaY_parser`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `expkind` | enum | Expression classification: constants (VK, VKNUM), variables (VLOCAL, VUPVAL, VGLOBAL), operations (VINDEXED, VJMP, VCALL), relocatability (VRELOCABLE, VNONRELOC) |
| `expdesc` | struct | Expression descriptor: kind, value union (int info/aux or lua_Number), patch lists for true/false control flow |
| `upvaldesc` | struct | Upvalue entry: kind flag and info index (2 bytes) |
| `FuncState` | struct | Compilation state for one function: code position, register allocation, constant/nested-function counts, local variable tracking, upvalue array, declaration stack |

## Global / File-Static State
None.

## Key Functions / Methods

### luaY_parser
- Signature: `Proto *luaY_parser(lua_State *L, ZIO *z, Mbuffer *buff, const char *name)`
- Purpose: Parser entry point; parses a Lua chunk from input stream and generates function prototype
- Inputs: Lua state, input stream (ZIO), work buffer, source name
- Outputs/Return: Parsed `Proto` (function prototype with bytecode, constants, nested functions)
- Side effects: Reads from stream, allocates memory, modifies buffer
- Calls: Not inferable from header (defined in lparser.c)
- Notes: Main public API; processes one top-level chunk

## Control Flow Notes
Not directly visible. Used during compilation phase to parse Lua source into bytecode. `FuncState` maintains a stack (via `prev` chain) for nested function scopes; parser recursively updates state as it encounters function definitions.

## External Dependencies
- **config.h**: Build configuration (HAVE_LUA guard)
- **llimits.h**: Type definitions (lu_byte, lu_mem, Instruction, LUAI_* constants)
- **lobject.h**: Lua runtime objects (Proto, Table, TValue, GCObject)
- **lzio.h**: Buffered I/O (ZIO, Mbuffer, lua_Reader callback type)
