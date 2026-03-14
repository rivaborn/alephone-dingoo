# Source_Files/Lua/lua_monsters.h

## File Purpose
Declares Lua C bindings for exposing the game's monster system to Lua scripts. Provides typed wrappers for individual monsters, monster collections, and monster action states using template-based classes that bridge C++ data structures with Lua.

## Core Responsibilities
- Define Lua class and container types for monsters and monster actions
- Declare string mnemonics for Lua class names ("monster", "Monsters", "monster_action")
- Register a function to initialize all monster-related Lua bindings with a Lua state
- Support type-safe access to monster data from Lua scripts using the template system from `lua_templates.h`

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Monster` | typedef (class template) | Type-safe wrapper for individual monster objects; uses `L_Class<>` template with int16 indices |
| `Lua_Monsters` | typedef (class template) | Container for all monsters; uses `L_Container<>` template to provide indexed access from Lua |
| `Lua_MonsterAction` | typedef (class template) | Enum wrapper for monster action states; uses `L_Enum<>` template to support mnemonic lookup |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Monster_Name[]` | char array | extern | Registered Lua class name "monster" used as metatable name |
| `Lua_Monsters_Name[]` | char array | extern | Registered Lua container name "Monsters" for global monster collection |
| `Lua_MonsterAction_Name[]` | char array | extern | Registered Lua enum name "monster_action" for action state enumeration |

## Key Functions / Methods

### Lua_Monsters_register
- **Signature:** `int Lua_Monsters_register(lua_State *L);`
- **Purpose:** Register all monster-related Lua bindings and metatables with the Lua state; called during engine initialization to make monsters accessible from scripts.
- **Inputs:** `L` ΓÇö the Lua state to register bindings into
- **Outputs/Return:** int (status code, likely 0 for success)
- **Side effects:** Modifies the Lua registry; registers metatables, methods, and the global `Monsters` container
- **Calls:** (implementation not visible; expected to call template `Register()` methods for each type)
- **Notes:** Not inferable from this header alone; implementation is in a corresponding `.cpp` file

## Control Flow Notes
This is a pure header file declaring interfaces. No control flow logic is present. The registration function is called during engine/Lua initialization to expose the monster system. At runtime, Lua scripts interact with monsters through the registered classes and container.

## External Dependencies
- **Lua C API headers:** `lua.h`, `lauxlib.h`, `lualib.h` (5.1)
- **Game engine headers:** `config.h`, `cseries.h`, `map.h`, `monsters.h` (for underlying data structures)
- **Template utilities:** `lua_templates.h` (defines `L_Class<>`, `L_Container<>`, `L_Enum<>` base templates)
- **Extern symbols used but not defined here:** Monster data structures and constants from `monsters.h` and `map.h`
