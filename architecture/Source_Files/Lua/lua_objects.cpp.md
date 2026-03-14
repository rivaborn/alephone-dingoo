# Source_Files/Lua/lua_objects.cpp

## File Purpose
Implements Lua C bindings for game world objects (effects, items, scenery, sounds) in Aleph One. Bridges Lua scripts to the native game engine, enabling level scripting to create and manipulate map objects.

## Core Responsibilities
- Register Lua classes/containers for Effects, Items, Scenery, Sounds with the Lua state
- Implement property accessors (position, facing, polygon, type) for each object type
- Provide object creation factories callable from Lua (`Effects.new()`, `Items.new()`, etc.)
- Handle object deletion and deanimation
- Validate object indices and enforce type safety
- Expose sound playback via object methods
- Maintain backward-compatibility wrapper functions for legacy Lua scripts

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Lua_Effect | typedef (L_Class) | Single effect instance wrapper |
| Lua_Effects | typedef (L_Container) | Container exposing effect collection to Lua |
| Lua_Item | typedef (L_Class) | Single item instance wrapper |
| Lua_Items | typedef (L_Container) | Item collection container |
| Lua_Scenery | typedef (L_Class) | Single scenery instance wrapper |
| Lua_Sceneries | typedef (L_Container) | Scenery collection container |
| Lua_Sound | typedef (L_LazyEnum) | Sound definition with lazy lookup |
| Lua_Sounds | typedef (L_EnumContainer) | Sound collection with mnemonic lookup |
| Lua_EffectType, Lua_ItemType, Lua_SceneryType | typedef (L_Enum) | Type enumerations for objects |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| AngleConvert | const float | file-static | Conversion factor: 360/FULL_CIRCLE for facing angles |
| Lua_Effect_Name | char[] | extern | Metatable name "effect" |
| Lua_Effects_Name | char[] | extern | Container name "Effects" |
| Lua_Item_Name, Lua_Items_Name, etc. | char[] | extern | Metatable/container names for each object type |
| Lua_Effect_Get | luaL_reg[] | file-static | Method table for effect property getters |
| Lua_Item_Get, Lua_Item_Set | luaL_reg[] | file-static | Getter/setter tables for items |
| Lua_Scenery_Get, Lua_Scenery_Set | luaL_reg[] | file-static | Getter/setter tables for scenery |
| compatibility_script | const char* | file-static | Legacy Lua function implementations for backward compatibility |

## Key Functions / Methods

### lua_delete_object (template)
- **Signature:** `template<class T> int lua_delete_object(lua_State *L)`
- **Purpose:** Remove an object from the map; specialized for Lua_Scenery to also deanimate
- **Inputs:** Lua state, object index (T::Index)
- **Outputs/Return:** 0 (no Lua return value)
- **Side effects:** Calls `remove_map_object()`; Lua_Scenery variant also calls `deanimate_scenery()`
- **Calls:** `remove_map_object()`, `deanimate_scenery()` (Scenery specialization)
- **Notes:** Template specialization guards Scenery-specific cleanup

### lua_play_object_sound (template)
- **Signature:** `template<class T> int lua_play_object_sound(lua_State *L)`
- **Purpose:** Play a sound effect from an object's location
- **Inputs:** Object index (arg 1), sound code from Lua_Sound (arg 2)
- **Outputs/Return:** 0
- **Side effects:** Engine sound playback
- **Calls:** `play_object_sound()`

### lua_object_position (template)
- **Signature:** `template<class T> int lua_object_position(lua_State *L)`
- **Purpose:** Set object world coordinates and polygon
- **Inputs:** x, y, z (world units, scaled by WORLD_ONE), polygon index (numeric or Lua_Polygon)
- **Outputs/Return:** 0
- **Side effects:** Updates object location; updates polygon object list if polygon changed
- **Calls:** `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`
- **Notes:** Validates polygon bounds; handles both numeric and enum polygon arguments

### get_object_x, get_object_y, get_object_z (template)
- **Signature:** `template<class T> static int get_object_{x,y,z}(lua_State *L)`
- **Purpose:** Return object position in world units (scaled by WORLD_ONE)
- **Inputs:** Object index
- **Outputs/Return:** 1 (pushes double to Lua)
- **Side effects:** None
- **Calls:** `get_object_data()`

### get_object_facing (template)
- **Signature:** `template<class T> static int get_object_facing(lua_State *L)`
- **Purpose:** Retrieve object facing angle in degrees (0ΓÇô360)
- **Inputs:** Object index
- **Outputs/Return:** 1 (double: facing * AngleConvert)
- **Calls:** `get_object_data()`

### set_object_facing (template)
- **Signature:** `template<class T> static int set_object_facing(lua_State *L)`
- **Purpose:** Set object facing angle
- **Inputs:** Object index, angle in degrees (converted back via AngleConvert)
- **Outputs/Return:** 0
- **Side effects:** Modifies object->facing
- **Notes:** Type-checks argument; returns error if not numeric

### get_object_polygon (template)
- **Signature:** `template<class T> static int get_object_polygon(lua_State *L)`
- **Purpose:** Push Lua_Polygon wrapper for object's containing polygon
- **Inputs:** Object index
- **Outputs/Return:** 1
- **Calls:** `get_object_data()`, `Lua_Polygon::Push()`

### get_object_type (template)
- **Signature:** `template<class T, class TT> static int get_object_type(lua_State *L)`
- **Purpose:** Push type enum (Lua_EffectType, Lua_ItemType, Lua_SceneryType) for object
- **Inputs:** Object index
- **Outputs/Return:** 1
- **Calls:** `get_object_data()`, `TT::Push()` (type-dependent push)

### Lua_Scenery_Damage
- **Signature:** `int Lua_Scenery_Damage(lua_State *L)`
- **Purpose:** Apply damage to scenery (triggers breaking/animation)
- **Inputs:** Scenery index
- **Outputs/Return:** 0
- **Side effects:** Calls `damage_scenery()`

### Lua_Scenery_Get_Damaged
- **Signature:** `static int Lua_Scenery_Get_Damaged(lua_State *L)`
- **Purpose:** Check if scenery is damaged (owner is _object_is_normal)
- **Inputs:** Scenery index
- **Outputs/Return:** 1 (boolean)
- **Notes:** Uses GET_OBJECT_OWNER macro to distinguish damaged vs. undamaged scenery

### Lua_Scenery_Get_Solid
- **Signature:** `static int Lua_Scenery_Get_Solid(lua_State *L)`
- **Purpose:** Check if scenery is solid (blocks collision)
- **Inputs:** Scenery index
- **Outputs/Return:** 1 (boolean)
- **Calls:** OBJECT_IS_SOLID macro

### Lua_Scenery_Set_Solid
- **Signature:** `static int Lua_Scenery_Set_Solid(lua_State *L)`
- **Purpose:** Toggle scenery solidity
- **Inputs:** Scenery index, boolean value
- **Outputs/Return:** 0
- **Side effects:** Modifies object solidity via SET_OBJECT_SOLIDITY macro

### Lua_Effects_New
- **Signature:** `static int Lua_Effects_New(lua_State *L)`
- **Purpose:** Create a new effect (particle/visual) in the world
- **Inputs:** x, y, z (doubles), polygon (numeric or Lua_Polygon), effect_type
- **Outputs/Return:** 1 (Lua_Effect or nil if creation failed)
- **Side effects:** Calls `::new_effect()`; allocates effect object on success
- **Calls:** `Lua_Polygon::Valid()`, `Lua_Polygon::Index()`, `Lua_EffectType::ToIndex()`, `::new_effect()`, `Lua_Effect::Push()`
- **Notes:** Returns nil if effect index is NONE; validates polygon index

### Lua_Items_New
- **Signature:** `int Lua_Items_New(lua_State *L)`
- **Purpose:** Create a new item (weapon, powerup, etc.) in the world
- **Inputs:** x, y, z, polygon, item_type (signature identical to Effects)
- **Outputs/Return:** 1 (Lua_Item or nil if creation failed)
- **Side effects:** Calls `::new_item()`
- **Calls:** Similar to Lua_Effects_New but uses `object_location` struct and `::new_item()`

### Lua_Sceneries_New
- **Signature:** `static int Lua_Sceneries_New(lua_State *L)`
- **Purpose:** Create a new scenery object (environmental prop) in the world
- **Inputs:** x, y, z, polygon, scenery_type
- **Outputs/Return:** 1 (Lua_Scenery or nil)
- **Side effects:** Calls `::new_scenery()`, `randomize_scenery_shape()`
- **Calls:** `Lua_SceneryType::ToIndex()`, `::new_scenery()`, `randomize_scenery_shape()`, `Lua_Scenery::Push()`
- **Notes:** Randomizes shape on creation for visual variety

### Lua_Scenery_Valid
- **Signature:** `static bool Lua_Scenery_Valid(int32 index)`
- **Purpose:** Validate scenery index, excluding player legs/torso
- **Inputs:** Index into objects array
- **Outputs/Return:** true if valid scenery (not occupied by player)
- **Side effects:** None
- **Calls:** `GetMemberWithBounds()`, `get_player_data()`, `get_monster_data()`, `get_object_data()`
- **Notes:** Iterates player list to exclude player monsters; checks parasitic_object field

### Lua_ItemType_Get_Ball
- **Signature:** `static int Lua_ItemType_Get_Ball(lua_State *L)`
- **Purpose:** Check if item type is a ball
- **Inputs:** Item type index
- **Outputs/Return:** 1 (boolean)
- **Calls:** `get_item_kind()`

### Lua_Sounds_New
- **Signature:** `static int Lua_Sounds_New(lua_State *L)`
- **Purpose:** Create a custom sound definition from file paths
- **Inputs:** Variable number of string arguments (up to 5 file paths)
- **Outputs/Return:** 1 (Lua_Sound or nil if index < 0)
- **Side effects:** Calls SoundManager to allocate and populate sound definition
- **Calls:** `SoundManager::instance()->NewCustomSoundDefinition()`, `AddCustomSoundSlot()`, `Lua_Sound::Push()`
- **Notes:** Stops after 5 slots; returns nil if SoundManager fails

### Lua_Objects_register
- **Signature:** `int Lua_Objects_register(lua_State *L)`
- **Purpose:** Entry point; registers all object classes, containers, and types with Lua
- **Inputs:** Lua state
- **Outputs/Return:** 0
- **Side effects:** Sets up Lua registry with metatables, method tables, and validation callbacks
- **Calls:** `Register()` methods for each Lua class/container; `boost::bind()` to wire up dynamic limits
- **Notes:** Also runs compatibility scripts for legacy Lua functions; sets Valid function pointers

### compatibility (static)
- **Signature:** `static void compatibility(lua_State *L)`
- **Purpose:** Execute backward-compatibility Lua script defining old function names
- **Inputs:** Lua state
- **Outputs/Return:** None
- **Side effects:** Loads and executes compatibility_script string
- **Calls:** `luaL_loadbuffer()`, `lua_pcall()`
- **Notes:** Wraps Items.new() under old names like `new_item()` and `get_item_polygon()`

## Control Flow Notes

**Initialization (engine startup):**
- `Lua_Objects_register()` is called during Lua environment setup to register all object bindings
- Metatables and method tables are pushed into the Lua registry
- Valid function pointers are initialized to enable type checking
- Legacy compatibility functions are executed to support old scripts

**Runtime (Lua script execution):**
- Scripts call `Effects.new()`, `Items.new()`, `Scenery.new()`, or `Sounds.new()` to create objects
- Scripts access object properties via getters (`.x`, `.y`, `.facing`, `.polygon`, `.type`, etc.)
- Scripts may modify limited properties via setters (`.facing`, `.solid` for scenery)
- Object deletion via `.delete()` removes from map and updates animation state

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` (Lua 5.0ΓÇô5.1 API)
- **Game world:** `effects.h`, `items.h`, `monsters.h`, `scenery.h`, `player.h`, `scenery_definitions.h` (object definitions and management)
- **Sound system:** `SoundManager.h` (custom sound registration)
- **Lua engine bindings:** `lua_map.h`, `lua_templates.h` (template base classes and polygon references)
- **Build system:** `config.h` (conditional compilation HAVE_LUA)
- **Boost:** `boost/bind.hpp` (function binding for dynamic limit callbacks)
- **Defined elsewhere:** `get_object_data()`, `remove_map_object()`, `new_effect()`, `new_item()`, `new_scenery()`, `play_object_sound()`, `damage_scenery()`, `deanimate_scenery()`, `randomize_scenery_shape()`, `get_dynamic_limit()`, `add_object_to_polygon_object_list()`, `remove_object_from_polygon_object_list()`
