# Source_Files/Lua/lua_player.h

## File Purpose
Declares the Lua scripting interface for player objects in Aleph One. Provides template-based bindings allowing Lua scripts to access individual players and iterate over the players collection via `Lua_Player` and `Lua_Players` classes.

## Core Responsibilities
- Define `Lua_Player` class wrapping player entity access for Lua
- Define `Lua_Players` container for iteration over all player objects
- Export the player registration function to set up Lua bindings
- Guard Lua-specific code behind compile-time feature flag

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Lua_Player | typedef (template specialization) | Wraps player indices for Lua access; instantiates `L_Class<Lua_Player_Name>` with `int16` index type |
| Lua_Players | typedef (template specialization) | Container for accessing all players; instantiates `L_Container<Lua_Players_Name, Lua_Player>` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Lua_Player_Name | extern char[] | global | String constant "player"; used as metatable/type name in Lua registry |
| Lua_Players_Name | extern char[] | global | String constant "Players"; used as global table name in Lua |

## Key Functions / Methods

### Lua_Player_register
- Signature: `int Lua_Player_register(lua_State *L);`
- Purpose: Initialize player class bindings in Lua state, registering metatables and methods
- Inputs: `lua_State *L` ΓÇô pointer to Lua interpreter state
- Outputs/Return: `int` ΓÇô status code (implementation in corresponding .cpp)
- Side effects: Modifies Lua registry; establishes `Players` global table and player accessor methods
- Calls: Not visible (delegated to .cpp implementation)
- Notes: Called once during engine Lua subsystem initialization

## Control Flow Notes
This header is included by the engine's Lua module during compilation. At runtime, `Lua_Player_register()` is invoked during Lua state creation to expose player objects. Scripts then access players via `Players[index]` or iterate the collection.

## External Dependencies
- **Lua 5.1**: `lua.h`, `lauxlib.h`, `lualib.h` (core C API)
- **Internal**: `lua_templates.h` (provides `L_Class<>` and `L_Container<>` template infrastructure)
- **Configuration**: `config.h` (compile-time `HAVE_LUA` guard)

## Notes
- Implementation is entirely template-driven via `lua_templates.h`; this header declares only the specializations and registration entry point
- Actual getter/setter methods registered elsewhere (likely in corresponding .cpp file)
- The `extern char` declarations allow the type names to be shared across translation units
