# Source_Files/Lua/lua_hud_objects.h

## File Purpose
Declares Lua C bindings for HUD (Heads-Up Display) objects and enums in the Aleph One game engine. Provides the glue layer allowing Lua scripts to interact with player, game, and screen HUD state via templated Lua class/enum wrappers.

## Core Responsibilities
- Define Lua class proxies (`Lua_HUDPlayer`, `Lua_HUDGame`, `Lua_HUDScreen`) for HUD game objects
- Define Lua enum proxies (`Lua_InventorySection`, `Lua_RendererType`, `Lua_SensorBlipType`, `Lua_TextureType`) for game enums
- Define Lua container types (`Lua_InventorySections`, `Lua_RendererTypes`, `Lua_SensorBlipTypes`, `Lua_TextureTypes`) for enum collections
- Expose a registration function to bind all these types to a Lua runtime
- Support Lua scripts reading/writing HUD state through the Lua stack

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Lua_HUDPlayer` | class typedef | Lua wrapper (via `L_Class` template) for player HUD object |
| `Lua_HUDGame` | class typedef | Lua wrapper for game HUD state |
| `Lua_HUDScreen` | class typedef | Lua wrapper for screen/rendering HUD state |
| `Lua_InventorySection` | enum typedef | Lua enum wrapper for inventory slot types |
| `Lua_InventorySections` | enum container typedef | Lua table-like container for all inventory section enums |
| `Lua_RendererType` | enum typedef | Lua enum wrapper for renderer backend types |
| `Lua_RendererTypes` | enum container typedef | Container for renderer type enums |
| `Lua_SensorBlipType` | enum typedef | Lua enum wrapper for motion sensor blip kinds |
| `Lua_SensorBlipTypes` | enum container typedef | Container for sensor blip type enums |
| `Lua_TextureType` | enum typedef | Lua enum wrapper for texture kinds (5 types) |
| `Lua_TextureTypes` | enum container typedef | Container for texture type enums |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_HUDPlayer_Name` | `char[]` (extern) | global | Metatable name string ("Player") for Lua class registration |
| `Lua_HUDGame_Name` | `char[]` (extern) | global | Metatable name string ("Game") for Lua class registration |
| `Lua_HUDScreen_Name` | `char[]` (extern) | global | Metatable name string ("Screen") for Lua class registration |
| `Lua_InventorySection_Name` | `char[]` (extern) | global | Enum name string ("inventory_section") |
| `Lua_InventorySections_Name` | `char[]` (extern) | global | Container name string ("InventorySections") |
| `Lua_RendererType_Name` | `char[]` (extern) | global | Enum name string ("renderer_type") |
| `Lua_RendererTypes_Name` | `char[]` (extern) | global | Container name string ("RendererTypes") |
| `Lua_SensorBlipType_Name` | `char[]` (extern) | global | Enum name string ("sensor_blip") |
| `Lua_SensorBlipTypes_Name` | `char[]` (extern) | global | Container name string ("SensorBlipTypes") |
| `Lua_TextureType_Name` | `char[]` (extern) | global | Enum name string ("texture_type") |
| `Lua_TextureTypes_Name` | `char[]` (extern) | global | Container name string ("TextureTypes") |
| `NUMBER_OF_LUA_TEXTURE_TYPES` | `#define` (5) | macro | Count of supported texture types for bounds checking |

## Key Functions / Methods

### Lua_HUDObjects_register
- **Signature:** `int Lua_HUDObjects_register(lua_State *L);`
- **Purpose:** Primary entry point to register all HUD object and enum bindings with a Lua runtime. Called once during game initialization.
- **Inputs:** `lua_State *L` ΓÇö active Lua interpreter state
- **Outputs/Return:** `int` ΓÇö Lua error code (0 on success, non-zero on failure)
- **Side effects (global state, I/O, alloc):** Modifies Lua registry; allocates Lua metatables and lookup tables for all class/enum types
- **Calls:** Not visible in this file; implementation is in corresponding `.cpp` file. Likely calls `L_Class<>::Register()`, `L_Enum<>::Register()`, and `L_EnumContainer<>::Register()` for each type.
- **Notes:** Must be called before Lua scripts can access HUD objects. Implementation likely iterates over class/enum typedefs and registers them with get/set method tables.

## Control Flow Notes

This is a **header-only declarations file**. It does not define control flow; the implementation is in a paired `.cpp` file. Expected flow at engine init:
1. Engine calls `Lua_HUDObjects_register(L)` to bind HUD types
2. Lua scripts can then instantiate or query HUD objects (e.g., `player = Player(0)`, `Game.ticks`)
3. The `L_Class` and `L_Enum` templates (from `lua_templates.h`) handle the metatable mechanics: `__index`, `__newindex`, enum equality, mnemonic lookup, etc.
4. Lua-side changes to HUD properties trigger C getter/setter methods, which read/write engine state

## External Dependencies

- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö Lua 5.1 stack manipulation, type checking, metatable registration
- **Template library:** `lua_templates.h` ΓÇö `L_Class<>`, `L_Enum<>`, `L_EnumContainer<>` template class definitions for Lua bindings
- **Game headers:** `items.h`, `map.h` ΓÇö Likely provide game object definitions referenced by HUD wrappers (defined elsewhere)
- **Platform layer:** `cseries.h` ΓÇö Cross-platform macros and types (SDL, endianness, etc.)
- **Config:** `config.h` ΓÇö Feature flags (e.g., `HAVE_LUA`, `HAVE_OPENGL`)
