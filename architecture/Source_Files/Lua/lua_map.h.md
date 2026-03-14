# Source_Files/Lua/lua_map.h

## File Purpose
Declares Lua bindings for game world map structures in Aleph One. Provides wrapper classes that expose C++ map geometry, lighting, and object data to Lua scripting, enabling level designers and scripters to manipulate map elements dynamically.

## Core Responsibilities
- Declare `L_Class`, `L_Container`, and `L_Enum` template instantiations for map-related types
- Export extern collection names used as template parameters for Lua binding registration
- Define typedef pairs combining template base classes with collection-specific names
- Declare the registration function that wires all map bindings into a Lua state

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Collection` | typedef (L_Enum) | Enumeration wrapper for collection IDs |
| `Lua_Collections` | typedef (L_EnumContainer) | Container for all collections |
| `Lua_ControlPanelClass` | typedef (L_Enum) | Enumeration for control panel classes |
| `Lua_ControlPanelType` | typedef (L_Enum) | Enumeration for control panel types |
| `Lua_DamageType` | typedef (L_Enum) | Enumeration for damage types |
| `Lua_Line` | typedef (L_Class) | Map line segment geometry |
| `Lua_Lines` | typedef (L_Container) | Container for all map lines |
| `Lua_Polygon` | typedef (L_Class) | Map polygon (area) |
| `Lua_Polygons` | typedef (L_Container) | Container for all polygons |
| `Lua_Platform` | typedef (L_Class) | Animated moving platform |
| `Lua_Platforms` | typedef (L_Container) | Container for platforms |
| `Lua_Light` | typedef (L_Class) | Light source |
| `Lua_Lights` | typedef (L_Container) | Container for lights |
| `Lua_Side` | typedef (L_Class) | Wall side (part of a line between polygons) |
| `Lua_Sides` | typedef (L_Container) | Container for sides |
| `Lua_Terminal` | typedef (L_Class) | Interactive terminal/computer |
| `Lua_Terminals` | typedef (L_Container) | Container for terminals |
| `Lua_TransferMode` | typedef (L_Enum) | Enumeration for rendering transfer modes |
| `Lua_Media` | typedef (L_Class) | Liquid/lava media |
| `Lua_Medias` | typedef (L_Container) | Container for media |
| `Lua_Tag` | typedef (L_Class) | Map tag/trigger |
| `Lua_Tags` | typedef (L_Container) | Container for tags |
| `Lua_PolygonCeiling`, `Lua_PolygonFloor` | typedef (L_Class) | Ceiling/floor surfaces of polygons |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Collection_Name` | extern char[] | global | String identifier "collection" |
| `Lua_Collections_Name` | extern char[] | global | String identifier "Collections" |
| `Lua_ControlPanelClass_Name` | extern char[] | global | String identifier "control_panel_class" |
| `Lua_ControlPanelType_Name` | extern char[] | global | String identifier "control_panel_type" |
| `Lua_DamageType_Name` | extern char[] | global | String identifier "damage_type" |
| `Lua_Line_Name` | extern char[] | global | String identifier "line" |
| `Lua_Lines_Name` | extern char[] | global | String identifier "Lines" |
| `Lua_Polygon_Name` | extern char[] | global | String identifier "polygon" |
| `Lua_Polygons_Name` | extern char[] | global | String identifier "Polygons" |
| `Lua_Platform_Name` | extern char[] | global | String identifier "platform" |
| `Lua_Light_Name` | extern char[] | global | String identifier "light" |
| `Lua_Lights_Name` | extern char[] | global | String identifier "Lights" |
| `Lua_TransferMode_Name` | extern char[] | global | String identifier "transfer_mode" |
| `Lua_Side_Name` | extern char[] | global | String identifier "side" |
| `Lua_Sides_Name` | extern char[] | global | String identifier "Sides" |
| `Lua_Media_Name` | extern char[] | global | String identifier "media" |
| `Lua_Terminal_Name` | extern char[] | global | String identifier "terminal" |
| `Lua_Tag_Name` | extern char[] | global | String identifier "tag" |

(Similar pattern for all other collection names; omitted for brevity)

## Key Functions / Methods

### Lua_Map_register
- **Signature:** `int Lua_Map_register(lua_State *L);`
- **Purpose:** Entry point to register all map-related Lua bindings with a Lua state.
- **Inputs:** `lua_State *L` ΓÇô Lua state to register bindings into.
- **Outputs/Return:** Integer (presumed Lua error code or count of registered items).
- **Side effects:** Modifies Lua registry; creates metatables and global tables for all map types.
- **Calls:** Not visible in this file; implementation elsewhere.
- **Notes:** Called during engine initialization to expose map API to Lua scripts. Declaration only; definition in corresponding `.cpp` file.

## Control Flow Notes
This header participates in the **initialization phase** of the engine. During startup, `Lua_Map_register()` is invoked to prepare the Lua scripting environment with map-related classes and containers. Lua scripts can then query and manipulate map geometry, lights, platforms, and other objects. No frame-by-frame or render-specific logic is present.

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô Core Lua embedding interfaces.
- **Engine templates:** `lua_templates.h` ΓÇô Template definitions for `L_Class`, `L_Container`, `L_Enum`, `L_EnumContainer`.
- **Game world structs:** `map.h` ΓÇô Core map data structures (polygons, lines, sides, objects, etc.).
- **Lighting system:** `lightsource.h` ΓÇô Light source definitions and accessors.
- **Build config:** `config.h` ΓÇô Conditional compilation; `HAVE_LUA` gate.
- **C++ series library:** `cseries.h` ΓÇô Common types and macros.
