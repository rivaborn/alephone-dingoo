# Source_Files/Lua/lua_objects.h

## File Purpose

Declares Lua 5.1 bindings for game world object types (effects, items, scenery, sounds). Uses C++ template metaprogramming to expose game engine objects to the Lua scripting subsystem. This header bridges the game engine and Lua VM by defining class wrappers and container types.

## Core Responsibilities

- Declare extern name strings for each Lua-exposed object type ("effect", "item", etc.)
- Define template-based Lua class wrappers for game objects (effects, items, scenery, sounds)
- Define enum types for enumerating effect and item type codes
- Define container types for iterating collections of objects in Lua
- Expose module registration entry point (Lua_Objects_register) for VM initialization
- Guard all declarations behind HAVE_LUA to support optional Lua integration

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Lua_Effect_Name | extern char[] | String identifier "effect" for Lua metatable |
| Lua_Effect | typedef L_Class | Lua userdata wrapper for a single effect object |
| Lua_Effects | typedef L_Container | Lua collection/table for iterating all effects |
| Lua_EffectType | typedef L_Enum | Lua enum for effect type codes with mnemonics |
| Lua_EffectTypes | typedef L_EnumContainer | Lua collection of effect types (by number or name) |
| Lua_Item | typedef L_Class | Lua userdata wrapper for a single item object |
| Lua_Items | typedef L_Container | Lua collection/table for iterating all items |
| Lua_ItemType | typedef L_Enum | Lua enum for item type codes with mnemonics |
| Lua_ItemTypes | typedef L_EnumContainer | Lua collection of item types (by number or name) |
| Lua_Scenery | typedef L_Class | Lua userdata wrapper for scenery objects |
| Lua_Sceneries | typedef L_Container | Lua collection for iterating scenery |
| Lua_Sound | typedef L_LazyEnum | Lazy-loaded enum for sound types (validation deferred) |
| Lua_Sounds | typedef L_EnumContainer | Lua collection of sounds by number or name |

## Global / File-Static State

None.

## Key Functions / Methods

### Lua_Objects_register
- Signature: `int Lua_Objects_register(lua_State *L);`
- Purpose: Register all game object types (effects, items, scenery, sounds) with the Lua VM
- Inputs: Lua VM state pointer
- Outputs/Return: Integer status code (0=success by Lua convention)
- Side effects: Populates Lua registry with metatables, instance tables, and method tables for each object type; calls L_Class<>::Register and L_Container<>::Register for each typedef
- Calls: (defined in accompanying .cpp file; not visible here)
- Notes: Entry point called during engine initialization to expose game objects to Lua scripts

## Control Flow Notes

Header-only declarations file. No execution flow logic. Used during module initialization: the game engine calls `Lua_Objects_register(L)` once at startup, which configures Lua metatables and method lookup tables. At runtime, Lua scripts access game objects via the registered names (e.g., `Effects`, `Items`, `EffectTypes`), which invoke C functions defined by the L_Class/L_Container templates.

## External Dependencies

- **Lua C API** (lua.h, lauxlib.h, lualib.h): Lua 5.1 core interpreter; stack operations, metatables, type checking
- **lua_templates.h**: Template base classes (L_Class, L_Enum, L_Container, L_LazyEnum, L_EnumContainer) that implement the Lua/C++ bridge mechanism
- **items.h**: Game item type definitions and prototypes (for Lua_Item context)
- **map.h**: Game world structures and object management (for Lua_Effect, Lua_Scenery context)
- **cseries.h**: Cross-platform compatibility and standard utilities
- **config.h**: Compile-time configuration (HAVE_LUA feature flag)
