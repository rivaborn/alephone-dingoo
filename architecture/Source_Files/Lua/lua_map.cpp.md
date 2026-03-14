# Source_Files/Lua/lua_map.cpp

## File Purpose
Implements Lua bindings for map-related game objects in the Aleph One game engine. Exposes collections, polygons, sides, lines, endpoints, platforms, lights, tags, media, annotations, fog, and level metadata to Lua scripts through getter/setter methods and container classes.

## Core Responsibilities
- Register Lua classes and containers for map geometry (polygons, lines, endpoints, sides)
- Implement property accessors (getters/setters) for map entity attributes (heights, textures, states)
- Expose dynamic world state (platforms, lights, tags, media) to Lua scripting
- Provide compatibility wrappers for old Lua scripts via embedded Lua code
- Bridge C++ map data structures with Lua's type system using template-based wrappers

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Polygon` | L_Class typedef | Polygon map geometry wrapper |
| `Lua_Line` | L_Class typedef | Line (wall) geometry wrapper |
| `Lua_Endpoint` | L_Class typedef | 2D vertex/endpoint wrapper |
| `Lua_Platform` | L_Class typedef | Dynamic platform state wrapper |
| `Lua_Side` | L_Class typedef | Wall surface (with textures/panels) wrapper |
| `Lua_Light` | L_Class typedef | Light source state wrapper |
| `Lua_Tag` | L_Class typedef | Switchable tagged entity wrapper |
| `Lua_Media` | L_Class typedef | Liquid/media volume wrapper |
| `Lua_Annotation` | L_Class typedef | Map annotation (editor notes) wrapper |
| `Lua_Fog` | L_Class typedef | Fog visual effect wrapper |
| `control_panel_definition` | struct | Local copy of control panel data (from devices.cpp) |
| `Lua_CompletionState` | L_Enum typedef | Level completion state enum |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Collection_Name` | `char[]` | extern | Mnemonic for collection class |
| `Lua_Collections_Name` | `char[]` | extern | Mnemonic for collections container |
| `Lua_ControlPanelType_Name` | `char[]` | extern | Mnemonic for control panel type enum |
| `Lua_Line_Name` | `char[]` | extern | Mnemonic for line class |
| `Lua_Platform_Name` | `char[]` | extern | Mnemonic for platform class |
| `Lua_Polygon_Name` | `char[]` | extern | Mnemonic for polygon class |
| `Lua_Light_Name` | `char[]` | extern | Mnemonic for light class |
| `Lua_Annotation_Name` | `char[]` | extern | Mnemonic for annotation class |
| `Lua_Fog_Name` | `char[]` | extern | Mnemonic for fog class |
| `compatibility_script` | `const char*` | static | Embedded Lua code for backward compatibility with old scripts |

## Key Functions / Methods

### Lua_Polygon_Floor_Get_Height
- Signature: `static int Lua_Polygon_Floor_Get_Height(lua_State *L)`
- Purpose: Return floor height of a polygon (converted from world units to Lua numbers)
- Inputs: Lua stack with polygon index at position 1
- Outputs/Return: 1 value pushed to Lua stack (floor height as double)
- Side effects: None (read-only)
- Calls: `get_polygon_data()`, `lua_pushnumber()`
- Notes: Height is divided by `WORLD_ONE` to convert from internal units

### Lua_Polygon_Floor_Set_Height
- Signature: `static int Lua_Polygon_Floor_Set_Height(lua_State *L)`
- Purpose: Modify floor height and recalculate dependent map geometry
- Inputs: Lua stack with polygon index (arg 1) and new height (arg 2)
- Outputs/Return: 0 (no return value)
- Side effects: Modifies `polygon->floor_height`, calls `recalculate_redundant_endpoint_data()` and `recalculate_redundant_line_data()` for all polygon vertices
- Calls: `get_polygon_data()`, `luaL_error()`, `recalculate_redundant_endpoint_data()`, `recalculate_redundant_line_data()`
- Notes: Cascades changes to all endpoints and lines in the polygon; multiplies by `WORLD_ONE` on input

### Lua_Platform_Get/Set_Active
- Signature: `static int Lua_Platform_Get_Active(lua_State *L)` and `static int Lua_Platform_Set_Active(lua_State *L)`
- Purpose: Read/write platform activation state
- Inputs: Lua stack with platform index; setter also takes boolean value
- Outputs/Return: 1 for getter (boolean), 0 for setter
- Side effects: Setter calls `set_platform_state()` with external game logic
- Calls: `get_platform_data()`, `PLATFORM_IS_ACTIVE()`, `set_platform_state()`, `lua_pushboolean()`, `luaL_error()`
- Notes: Platform state affects geometry visibility and player interaction

### Lua_Line_Get_Endpoints
- Signature: `static int Lua_Line_Get_Endpoints(lua_State *L)`
- Purpose: Return a container of the 2 endpoints for a line
- Inputs: Lua stack with line index at position 1
- Outputs/Return: 1 value (Lua_Line_Endpoints object)
- Side effects: None (read-only)
- Calls: `Lua_Line_Endpoints::Push()`, `Lua_Line::Index()`
- Notes: Endpoints container has custom metatable for 0/1 indexing

### Lua_Annotation_Set_Text
- Signature: `static int Lua_Annotation_Set_Text(lua_State *L)`
- Purpose: Update annotation text with bounds checking
- Inputs: Lua stack with annotation index (arg 1) and string (arg 2)
- Outputs/Return: 0 (no return value)
- Side effects: Modifies `MapAnnotationList[index].text` with null-termination
- Calls: `Lua_Annotation::Index()`, `luaL_error()`, `strncpy()`
- Notes: Safely truncates to `MAXIMUM_ANNOTATION_TEXT_LENGTH` and null-terminates

### Lua_Annotations_New
- Signature: `static int Lua_Annotations_New(lua_State *L)`
- Purpose: Create new map annotation and add to annotation list
- Inputs: Lua args: polygon (optional), text (required), x (optional), y (optional)
- Outputs/Return: 1 value (new Lua_Annotation object)
- Side effects: Appends to `MapAnnotationList`, increments `dynamic_world->default_annotation_count`, checks for overflow
- Calls: `Lua_Polygon::Index()`, `get_polygon_data()`, `luaL_optint()`, `strncpy()`, `Lua_Annotation::Push()`, `luaL_error()`
- Notes: Defaults x/y to polygon center if polygon provided; errors if annotation limit reached

### Lua_Tag_Get_Active
- Signature: `static int Lua_Tag_Get_Active(lua_State *L)`
- Purpose: Check if any light or platform with a given tag is active
- Inputs: Lua stack with tag ID at position 1
- Outputs/Return: 1 value (boolean)
- Side effects: None (read-only, but iterates all lights and platforms)
- Calls: `Lua_Tag::Index()`, `get_light_status()`, `PLATFORM_IS_ACTIVE()`, `lua_pushboolean()`
- Notes: Searches all lights and platforms; expensive O(n) operation

### Lua_Map_register
- Signature: `int Lua_Map_register(lua_State *L)`
- Purpose: Register all map-related Lua classes, containers, and enums
- Inputs: Lua state pointer
- Outputs/Return: 0 (standard Lua init return)
- Side effects: Modifies Lua registry with all class registrations, sets global `Level` userdata
- Calls: Multiple `Register()` calls for each class/container/enum, `Lua_Level::Push()`, `lua_setglobal()`, `compatibility()`
- Notes: This is the entry point for Lua map bindings; called during level initialization

### compatibility
- Signature: `static void compatibility(lua_State *L)`
- Purpose: Load embedded Lua code for backward compatibility with old scripts
- Inputs: Lua state pointer
- Outputs/Return: void
- Side effects: Executes Lua code defining wrapper functions that map old API to new API
- Calls: `luaL_loadbuffer()`, `lua_pcall()`
- Notes: Embedded script defines ~40 compatibility functions like `get_polygon_floor_height()`, `set_platform_state()`, etc.

## Control Flow Notes
This file is **initialization-time binding code**. `Lua_Map_register()` is called once when the Lua subsystem initializes for a level. It does not participate in the frame/update loop. Runtime access to these bound objects occurs when Lua scripts call getter/setter methods, which invoke the static functions registered here. Property changes that modify geometry (e.g., height changes) trigger recalculation functions to maintain map invariants. Compatibility functions wrap new API calls to provide backward compatibility with older script code.

## External Dependencies
- **Lua headers**: `lua.h`, `lauxlib.h`, `lualib.h` (C Lua API)
- **Lua template framework**: `lua_templates.h` (defines `L_Class<>`, `L_Enum<>`, `L_Container<>`)
- **Map geometry**: `map.h` (polygon, line, endpoint, side data structures)
- **Dynamic data**: `lightsource.h`, `media.h`, `platforms.h` (dynamic world state)
- **Support headers**: `lua_monsters.h`, `lua_objects.h` (cross-subsystem Lua bindings)
- **Engine systems**: `OGL_Setup.h` (fog rendering), `SoundManager.h` (audio control)
- **Boost**: `boost/bind.hpp` (function binding, used in comments but not actively in this file)
- **Defined elsewhere**: `get_collection_definition()`, `get_panel_class()`, `get_control_panel_definition()`, `get_endpoint_data()`, `get_line_data()`, `get_platform_data()`, `get_polygon_data()`, `get_light_data()`, `get_media_data()`, `set_platform_state()`, `adjust_platform_*()`, `set_light_status()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `assume_correct_switch_position()`, `calculate_level_completion_state()`, `OGL_GetFogData()`, `number_of_terminal_texts()` (all accessed from external modules)
