# Source_Files/Lua/lua_projectiles.cpp

## File Purpose
Implements Lua bindings for the projectile system in Aleph One (Marathon-compatible game engine). Exposes projectile properties, methods, and type information to Lua scripts, enabling script-driven projectile spawning and manipulation.

## Core Responsibilities
- Expose projectile entity properties (position, damage, owner, target, angles, type) as Lua-accessible fields
- Provide methods for projectile manipulation (positioning, sound playback)
- Create and manage projectile instances from Lua
- Register projectile types and damage information enums
- Handle unit conversions between internal engine units and Lua-facing degrees/floating-point
- Maintain compatibility layer for legacy Lua script APIs

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `projectile_data` | struct (C++) | Internal projectile state; accessed via `get_projectile_data()` |
| `object_data` | struct (C++) | Shared object representation (position, facing, polygon); accessed via `get_object_data()` |
| `world_point3d` | struct (C++) | 3D coordinate; used for position/vector |
| `projectile_definition` | struct (C++) | Projectile type metadata; accessed via `get_projectile_definition()` |
| `Lua_Projectile` | L_Class | Lua wrapper for individual projectile entity |
| `Lua_Projectiles` | L_Container | Lua table-like container for projectile collection |
| `Lua_ProjectileType` | L_Enum | Lua enum for projectile type IDs with mnemonics |
| `Lua_ProjectileTypeDamage` | L_Class | Lua wrapper for damage info attached to a projectile type |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AngleConvert` | float constant | static | Conversion factor (360 / FULL_CIRCLE) for angle unit conversion |
| `Lua_Projectile_Name` | char[] | global | String literal "projectile" used as Lua class name |
| `Lua_Projectiles_Name` | char[] | global | String literal "Projectiles" used as Lua container name |
| `Lua_ProjectileType_Name` | char[] | global | String literal "projectile_type" |
| `Lua_ProjectileTypes_Name` | char[] | global | String literal "ProjectileTypes" |
| `Lua_ProjectileTypeDamage_Name` | char[] | global | String literal "projectile_type_damage" |
| `Lua_Projectile_Get[]` | luaL_reg[] | static | Method dispatch table for property getters |
| `Lua_Projectile_Set[]` | luaL_reg[] | static | Method dispatch table for property setters |
| `Lua_Projectiles_Methods[]` | luaL_reg[] | static | Method dispatch table for Projectiles container |
| `Lua_ProjectileType_Get[]` | luaL_reg[] | static | Method dispatch table for ProjectileType getters |
| `Lua_ProjectileTypeDamage_Get[]` | luaL_reg[] | static | Method dispatch table for damage info getters |
| `compatibility_script` | const char[] | static | Embedded Lua code providing legacy API wrappers |

## Key Functions / Methods

### Lua_Projectile_Get_Damage_Scale
- Signature: `static int(lua_State *L)`
- Purpose: Retrieve projectile damage scale multiplier
- Inputs: `L[1]` = projectile userdata
- Outputs/Return: Pushes `damage_scale / FIXED_ONE` (double) to stack; returns 1
- Side effects: None
- Calls: `Lua_Projectile::Index()`, `get_projectile_data()`, `lua_pushnumber()`
- Notes: FIXED_ONE is used for fixed-point arithmetic in internal representation

### Lua_Projectile_Get_Elevation
- Signature: `static int(lua_State *L)`
- Purpose: Retrieve projectile pitch (elevation angle)
- Inputs: `L[1]` = projectile userdata
- Outputs/Return: Pushes elevation in degrees (normalized to 0ΓÇô360 range); returns 1
- Side effects: None
- Calls: `get_projectile_data()`, unit conversion via `AngleConvert`
- Notes: Converts from internal FULL_CIRCLE units; clamps negative angles by adding 360

### Lua_Projectile_Get_Facing
- Signature: `static int(lua_State *L)`
- Purpose: Retrieve projectile yaw (horizontal facing angle)
- Inputs: `L[1]` = projectile userdata
- Outputs/Return: Pushes facing in degrees; returns 1
- Side effects: None
- Calls: `get_projectile_data()`, `get_object_data()`, angle conversion
- Notes: Accesses object's facing field; stored in shared object_data struct

### Lua_Projectile_Get_Gravity
- Signature: `static int(lua_State *L)`
- Purpose: Retrieve projectile vertical velocity (gravity/dz)
- Inputs: `L[1]` = projectile userdata
- Outputs/Return: Pushes `gravity / WORLD_ONE` (double); returns 1
- Side effects: None
- Calls: `get_projectile_data()`, `lua_pushnumber()`
- Notes: WORLD_ONE scales internal world coordinates to floating-point

### Lua_Projectile_Get_Owner
- Signature: `static int(lua_State *L)`
- Purpose: Retrieve monster that fired this projectile
- Inputs: `L[1]` = projectile userdata
- Outputs/Return: Pushes `Lua_Monster` object or nil; returns 1
- Side effects: None
- Calls: `get_projectile_data()`, `Lua_Monster::Push()`
- Notes: Pushes nil-like Lua_Monster if owner_index is invalid

### Lua_Projectile_Get_Target
- Signature: `static int(lua_State *L)`
- Purpose: Retrieve intended target monster
- Inputs: `L[1]` = projectile userdata
- Outputs/Return: Pushes `Lua_Monster` object or nil; returns 1
- Side effects: None
- Calls: Similar to Get_Owner
- Notes: May be NONE (nil) if projectile is non-homing

### Lua_Projectile_Get_Polygon
- Signature: `static int(lua_State *L)`
- Purpose: Retrieve containing polygon
- Inputs: `L[1]` = projectile userdata
- Outputs/Return: Pushes `Lua_Polygon` object; returns 1
- Side effects: None
- Calls: `get_projectile_data()`, `get_object_data()`, `Lua_Polygon::Push()`

### Lua_Projectile_Get_Type
- Signature: `static int(lua_State *L)`
- Purpose: Retrieve projectile type enum
- Inputs: `L[1]` = projectile userdata
- Outputs/Return: Pushes `Lua_ProjectileType` enum; returns 1
- Side effects: None
- Calls: `get_projectile_data()`, `Lua_ProjectileType::Push()`

### Lua_Projectile_Get_X, _Y, _Z
- Signature: `static int(lua_State *L)` each
- Purpose: Retrieve position component
- Inputs: `L[1]` = projectile userdata
- Outputs/Return: Pushes position / WORLD_ONE (double); returns 1
- Side effects: None
- Calls: `get_projectile_data()`, `get_object_data()`
- Notes: Accesses object->location.[xyz] and scales to floating-point

### Lua_Projectile_Set_Damage_Scale
- Signature: `static int(lua_State *L)`
- Purpose: Modify projectile damage scale
- Inputs: `L[1]` = projectile userdata; `L[2]` = scale factor (number)
- Outputs/Return: Returns 0 (no return value)
- Side effects: Modifies projectile->damage_scale
- Calls: Type check, `get_projectile_data()`
- Notes: Converts from double to FIXED_ONE integer; errors if L[2] is not a number

### Lua_Projectile_Set_Elevation, Facing, Gravity
- Signature: `static int(lua_State *L)` each
- Purpose: Modify projectile angles or velocity
- Inputs: `L[1]` = projectile userdata; `L[2]` = value (number)
- Outputs/Return: Returns 0
- Side effects: Modifies internal angle/velocity fields with unit conversion
- Calls: Type check, accessor functions
- Notes: Set_Elevation normalizes negative angles; all perform unit conversions

### Lua_Projectile_Set_Owner / Set_Target
- Signature: `static int(lua_State *L)`
- Purpose: Change projectile owner or target
- Inputs: `L[1]` = projectile userdata; `L[2]` = monster (nil, number, Lua_Monster, or Lua_Player)
- Outputs/Return: Returns 0
- Side effects: Modifies projectile->owner_index or ->target_index
- Calls: Type dispatch (lua_isnil, lua_isnumber, Lua_Monster::Is, Lua_Player::Is), index extraction
- Notes: Handles polymorphic input (allows Lua_Player, extracts its monster_index); NONE constant for nil

### Lua_Projectile_Play_Sound
- Signature: `int(lua_State *L)`
- Purpose: Emit sound effect at projectile location
- Inputs: `L[1]` = projectile userdata; `L[2]` = sound enum/index
- Outputs/Return: Returns 0
- Side effects: Plays sound via audio system
- Calls: `Lua_Sound::ToIndex()`, `get_projectile_data()`, `play_object_sound()`
- Notes: Sound localized to projectile's object_index

### Lua_Projectile_Position
- Signature: `int(lua_State *L)`
- Purpose: Reposition projectile (move and/or change polygon)
- Inputs: `L[1]` = projectile userdata; `L[2..4]` = x, y, z (numbers); `L[5]` = polygon (number or Lua_Polygon)
- Outputs/Return: Returns 0
- Side effects: Updates object location; moves projectile in spatial polygon list if polygon changes
- Calls: Type checks, `get_projectile_data()`, `get_object_data()`, polygon validation, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`
- Notes: Maintains spatial coherency; only updates list if polygon_index differs

### Lua_Projectiles_New_Projectile
- Signature: `static int(lua_State *L)`
- Purpose: Factory: create and push new projectile
- Inputs: `L[1..3]` = x, y, z (numbers); `L[4]` = polygon (number or Lua_Polygon); `L[5]` = type (number or Lua_ProjectileType)
- Outputs/Return: Pushes new `Lua_Projectile` object; returns 1
- Side effects: Calls engine factory; allocates projectile slot
- Calls: Polygon validation, type conversion, `new_projectile()`, `Lua_Projectile::Push()`
- Notes: Initializes projectile with default velocity vector (1, 0, 0 in world units); owner, target set to NONE

### Lua_Projectile_Valid
- Signature: `bool(int32 index)`
- Purpose: Validate projectile index
- Inputs: `index` = candidate projectile index
- Outputs/Return: Returns true if index is in valid range and slot is in-use
- Side effects: None
- Calls: `GetMemberWithBounds()`, `SLOT_IS_USED()` macro
- Notes: Checks both bounds (0ΓÇôMAXIMUM_PROJECTILES_PER_MAP) and slot usage

### Lua_ProjectileType_Valid
- Signature: `static bool(int32 index)`
- Purpose: Validate projectile type index
- Inputs: `index` = candidate type index
- Outputs/Return: Returns true if 0 Γëñ index < NUMBER_OF_PROJECTILE_TYPES
- Side effects: None

### Lua_Projectiles_register
- Signature: `int(lua_State *L)`
- Purpose: Initialize all Lua projectile bindings (registration entry point)
- Inputs: `L` = Lua interpreter
- Outputs/Return: Returns 0
- Side effects: Registers all Lua classes, containers, enums, methods
- Calls: `Lua_Projectile::Register()`, `Lua_Projectiles::Register()`, `Lua_ProjectileType::Register()`, `Lua_ProjectileTypeDamage::Register()`, `Lua_ProjectileTypes::Register()`, setter callbacks (boost::bind), `compatibility()`
- Notes: Sets up validity callbacks, dynamic length getter, loads legacy compatibility layer

### compatibility (static helper)
- Signature: `static void(lua_State *L)`
- Purpose: Load legacy Lua API compatibility wrappers
- Inputs: `L` = Lua interpreter
- Outputs/Return: Void
- Side effects: Executes embedded Lua code defining old-style getter/setter functions
- Calls: `luaL_loadbuffer()`, `lua_pcall()`
- Notes: Maps old APIs (e.g., `get_projectile_owner()`) to new property-based interface

## Control Flow Notes
**Initialization:** `Lua_Projectiles_register()` is called once at engine startup, setting up the entire Lua projectile subsystem.

**Script Access:** Lua scripts access projectiles via:
- `Projectiles[index]` ΓÇö indexed access via container __index metamethod
- `.property` ΓÇö property getter/setter invocation
- `:method()` ΓÇö method call via __index and L_TableFunction wrapper

**Data Flow:** Getter/setter functions act as bridges: extract Lua values ΓåÆ access C++ engine data ΓåÆ perform unit conversions ΓåÆ push/apply results.

**Spatial Coherency:** Position changes automatically maintain the polygon spatial index by removing/re-adding to lists.

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h`
- **Boost:** `boost/bind.hpp` (function binding)
- **Engine data structures:** `projectile_data`, `object_data`, `player_data`, `projectile_definition` (from `projectiles.h`, `map.h`, `player.h`, etc.)
- **Engine factories/accessors:** `new_projectile()`, `get_projectile_data()`, `get_object_data()`, `get_player_data()`, `get_projectile_definition()`, `play_object_sound()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`
- **Other Lua bindings:** `Lua_Monster`, `Lua_Player`, `Lua_Polygon`, `Lua_Sound`, `Lua_DamageType`, `Lua_ProjectileType` (from included headers)
- **Lua template system:** `L_Class`, `L_Enum`, `L_Container`, `L_EnumContainer`, `L_TableFunction` (from `lua_templates.h`)
- **Engine limits:** `get_dynamic_limit(_dynamic_limit_projectiles)` from `dynamic_limits.h`
