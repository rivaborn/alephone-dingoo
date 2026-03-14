# Source_Files/Lua/lua_monsters.cpp

## File Purpose
Implements Lua bindings for monster objects, types, and collections in the game engine. Exposes C++ monster data structures and control functions to Lua scripts through template-based wrapper classes, enabling scripted monster behavior and configuration.

## Core Responsibilities
- Registers Lua classes for monsters, monster types, modes, classes, actions, and related attributes
- Provides getter/setter accessors for monster properties (position, facing, velocity, health, visibility)
- Implements monster action methods (pathfinding, attacking, damage, sound playback)
- Manages bitwise property accessors for relationships and damage types (enemies, friends, immunities, weaknesses)
- Loads a compatibility layer translating old Lua API calls to new bindings
- Validates monster and polygon indices at Lua boundaries

## Key Types / Data Structures
Template instantiations from `lua_templates.h` (all typedef'd):

| Name | Kind (struct/enum/class/typedef/interface/trait) | Purpose |
|------|----------|---------|
| `Lua_Monster` | `L_Class<>` typedef | Wraps individual monster instances with index access |
| `Lua_Monsters` | `L_Container<>` typedef | Collection accessor for all monsters on map |
| `Lua_MonsterType` | `L_Enum<>` typedef | Enum wrapper for monster type definitions |
| `Lua_MonsterClass` | `L_Enum<>` typedef | Enum wrapper for monster classification (flags, powers of 2) |
| `Lua_MonsterMode` | `L_Enum<>` typedef | Enum wrapper for monster AI state (locked, lost_lock, unlocked, etc.) |
| `Lua_MonsterAction` | `L_Enum<>` typedef | Enum wrapper for monster animation state (stationary, attacking, dying, etc.) |
| `Lua_MonsterType_Enemies`, `_Friends`, `_Immunities`, `_Weaknesses` | `L_Class<>` typedef | Accessors for bitmasked monster properties |
| `monster_pathfinding_data` | `struct` | Context for monster path queries (definition, monster, zone crossing flag) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_MonsterClass_Name` | `char[]` | global (external) | Metadata name string "monster_class" |
| `Lua_MonsterType_Name` | `char[]` | global (external) | Metadata name string "monster_type" |
| `Lua_Monster_Name` | `char[]` | global (external) | Metadata name string "monster" |
| `Lua_Monsters_Name` | `char[]` | global (external) | Metadata name string "Monsters" |
| `AngleConvert` | `float` (360/FULL_CIRCLE) | static | Unit conversion for angle representation |
| `compatibility_script` | `const char*` | static | Multiline Lua code string wrapping old API in new bindings |
| `Lua_Monster_Get[]` | `luaL_reg[]` | static | Method table for monster property getters |
| `Lua_Monster_Set[]` | `luaL_reg[]` | static | Method table for monster property setters |
| `Lua_MonsterType_Get[]`, `Lua_MonsterType_Set[]` | `luaL_reg[]` | static | Method tables for monster type accessors |
| Various `*_Metatable[]` | `luaL_reg[]` | static | Metamethods (__index, __newindex) for Lua property access |

## Key Functions / Methods

### Lua_Monsters_register
- **Signature:** `int Lua_Monsters_register(lua_State *L)`
- **Purpose:** Initializes and registers all monster-related Lua bindings with the Lua state
- **Inputs:** Lua state pointer
- **Outputs/Return:** 0 (success; errors via lua_error)
- **Side effects:** Creates Lua tables (MonsterClasses, MonsterTypes, MonsterModes, MonsterActions, Monsters); registers metatables; loads compatibility functions
- **Calls:** `Register()` on all L_Enum/L_EnumContainer/L_Class types; `compatibility(L)`
- **Notes:** Called once during engine initialization; sets Valid functors for type checking; uses `boost::bind` for dynamic monster limit

### Lua_Monsters_New
- **Signature:** `int Lua_Monsters_New(lua_State *L)` ΓÇö args: (x, y, z, polygon, type)
- **Purpose:** Creates a new monster instance at specified location and type
- **Inputs:** Lua numbers (x, y, z in world units), polygon index or Lua_Polygon, monster type
- **Outputs/Return:** Pushes Lua_Monster instance or nil; returns 1 (stack count)
- **Side effects:** Calls C++ `new_monster()`, allocates monster data slot
- **Calls:** `lua_tonumber()`, `Lua_Polygon::Valid/Index()`, `Lua_MonsterType::ToIndex()`, `::new_monster()`, `Lua_Monster::Push()`
- **Notes:** Returns nil and 0 on allocation failure; validates polygon index early

### Lua_Monster_Move_By_Path
- **Signature:** `int Lua_Monster_Move_By_Path(lua_State *L)` ΓÇö args: (monster, polygon_index)
- **Purpose:** Pathfinds a monster to a destination polygon and begins traversal
- **Inputs:** Lua monster, target polygon index or Lua_Polygon
- **Outputs/Return:** 0 on error/success (Lua convention)
- **Side effects:** Allocates/deallocates pathfinding structures; activates inactive monsters; modifies monster path and action fields
- **Calls:** `Lua_Monster::Index()`, `get_monster_data()`, `Lua_Polygon::Valid/Index()`, `get_monster_definition_external()`, `get_object_data()`, `activate_monster()`, `delete_path()`, `new_path()` (with callback), `advance_monster_path()`, `set_monster_action()`, `set_monster_mode()`
- **Notes:** Rejects player monsters; clears existing path before creating new one; sets cross_zone_boundaries=true; transitions to idle/unlocked if pathfinding fails

### Lua_Monster_Position
- **Signature:** `int Lua_Monster_Position(lua_State *L)` ΓÇö args: (monster, x, y, z, polygon)
- **Purpose:** Teleports a monster to a new position and polygon
- **Inputs:** Lua monster, world coordinates (x, y, z), destination polygon
- **Outputs/Return:** 0
- **Side effects:** Modifies object location; removes from old polygon's object list; adds to new polygon's object list
- **Calls:** `Lua_Monster::Index()`, `get_monster_data()`, `get_object_data()`, `Lua_Polygon::Valid/Index()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`
- **Notes:** Performs polygon relocation only if polygon index changes; coordinate scaling by WORLD_ONE

### Lua_Monster_Accelerate
- **Signature:** `int Lua_Monster_Accelerate(lua_State *L)` ΓÇö args: (monster, direction, velocity, vertical_velocity)
- **Purpose:** Applies directional acceleration and vertical velocity to a monster
- **Inputs:** Lua numbers for direction (degrees), velocity, vertical velocity (world units per tick)
- **Outputs/Return:** 0
- **Side effects:** Updates monster velocity state via C++ engine
- **Calls:** `accelerate_monster()` with angle conversion and unit scaling
- **Notes:** Converts degrees to internal angle units via AngleConvert; scales velocities by WORLD_ONE

### Lua_Monster_Damage
- **Signature:** `int Lua_Monster_Damage(lua_State *L)` ΓÇö args: (monster, amount[, damage_type])
- **Purpose:** Inflicts damage on a monster from Lua
- **Inputs:** Lua monster, damage amount (number), optional damage type
- **Outputs/Return:** 0
- **Side effects:** Modifies monster vitality and may trigger death/hit animations
- **Calls:** `Lua_Monster::Index()`, `get_monster_data()`, `Lua_DamageType::ToIndex()`, `damage_monster()` with constructed damage_definition
- **Notes:** Defaults to `_damage_fist` if type unspecified; damage flags and scale hardcoded; passes NONE for aggressor/projectile

### Lua_Monster_Attack
- **Signature:** `int Lua_Monster_Attack(lua_State *L)` ΓÇö args: (monster, target_index_or_monster)
- **Purpose:** Sets a monster's attack target
- **Inputs:** Lua monster, target (number index or Lua_Monster instance)
- **Outputs/Return:** 0 on error, returns 1 after success
- **Side effects:** Updates monster AI target via C++ engine
- **Calls:** `change_monster_target()`
- **Notes:** Accepts either numeric monster index or Lua_Monster object; validates with `Lua_Monster::Is()`

### Lua_Monster_Play_Sound
- **Signature:** `int Lua_Monster_Play_Sound(lua_State *L)` ΓÇö args: (monster, sound_index)
- **Purpose:** Plays a sound at the monster's current location
- **Inputs:** Lua monster, sound index
- **Outputs/Return:** 0
- **Side effects:** Triggers audio playback from object location
- **Calls:** `Lua_Sound::ToIndex()`, `get_monster_data()`, `get_object_data()`, `play_object_sound()`

### Lua_Monster property getters (X, Y, Z, Facing, Vitality, Visible, Active, Mode, Action, Type, Polygon, Player, External_Velocity, Vertical_Velocity)
- **Signature:** `static int Lua_Monster_Get_<property>(lua_State *L)`
- **Purpose:** Retrieve a monster property value
- **Inputs:** Lua monster at stack index 1
- **Outputs/Return:** Pushes value; returns 1
- **Calls:** `Lua_Monster::Index()`, `get_monster_data()`, `get_object_data()`, conversions via WORLD_ONE or AngleConvert
- **Notes:** Property-specific return types (number, boolean, enum); coordinate/angle scaling applied

### Lua_Monster property setters (Active, Facing, Visible, Vitality, External_Velocity, Vertical_Velocity)
- **Signature:** `static int Lua_Monster_Set_<property>(lua_State *L)`
- **Purpose:** Modify a monster property
- **Inputs:** Lua monster, new value
- **Outputs/Return:** 0
- **Side effects:** Modifies monster or object data; `Set_Visible` has conditional logic to prevent messing with active monsters
- **Notes:** Type checking via `lua_is*()` with error reporting; scaling applied; `Set_Visible` short-circuits if teleporting out

### Lua_MonsterType property accessors (Get_Class, Get_Height, Get_Friends, Get_Enemies, Get_Immunities, Get_Weaknesses, Get_Impact_Effect, Get_Item, Get_Radius, Set_Class, Set_Item)
- **Purpose:** Read/write monster type definition fields
- **Outputs/Return:** Pushes enum or number; returns 1 for getters
- **Calls:** `get_monster_definition_external()`, then push Lua types or numeric values
- **Notes:** Some return scaled values (height, radius); `Get_Friends/Enemies/Immunities/Weaknesses` return wrapper objects for indexed access

### Lua_MonsterType_Enemies/Friends/Immunities/Weaknesses Get/Set
- **Purpose:** Bitmasked property accessors (metamethod handlers for `[]` syntax)
- **Signature:** Static functions registered as `__index` / `__newindex` metamethods
- **Inputs:** Monster type, class/damage type, boolean value (for Set)
- **Outputs/Return:** Push boolean (Get); return 0 (Set)
- **Side effects:** Bitwise OR/AND operations on `definition->enemies/friends/immunities/weaknesses`
- **Notes:** Use power-of-two validation for classes; bitshift operations for damage types

### Lua_Monster_Valid
- **Signature:** `int Lua_Monster_Valid(int16 index)`
- **Purpose:** Validate a monster index
- **Inputs:** Monster index
- **Outputs/Return:** Boolean (true if index in range and slot used)
- **Calls:** `GetMemberWithBounds()`, `SLOT_IS_USED()` macro
- **Notes:** Bounds-checked via MAXIMUM_MONSTERS_PER_MAP

### compatibility
- **Signature:** `static void compatibility(lua_State *L)`
- **Purpose:** Load Lua compatibility wrapper functions
- **Inputs:** Lua state
- **Side effects:** Executes `compatibility_script` to define old-API functions (e.g., `activate_monster()`, `move_monster()`)
- **Calls:** `luaL_loadbuffer()`, `lua_pcall()`
- **Notes:** ~30 wrapper functions mapping old API to new bindings; includes unit conversions (e.g., 1024x scaling for height/position)

Trivial helpers (summarized in Notes):
- `powerOfTwo()` ΓÇö bitwise validity check
- L_Container iterator and length overrides
- Various `_Get`/`_Set` metamethod handlers for Lua property access

## Control Flow Notes
This file is part of the Lua scripting layer, not the main game loop. Initialization flow:
1. Engine calls `Lua_Monsters_register()` during startup
2. Lua scripts can then instantiate, query, and command monsters at any point
3. Actual monster updates (physics, AI, animation) happen in `monsters.cpp` via engine ticks
4. Compatibility wrappers allow legacy Lua maps to run without modification

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` (conditional on `HAVE_LUA`)
- **Templates:** `lua_templates.h` (L_Class, L_Enum, L_Container, L_EnumContainer)
- **Sibling Lua bindings:** `lua_map.h` (Lua_Polygon, Lua_DamageType), `lua_objects.h` (Lua_EffectType, Lua_Sound, Lua_ItemType), `lua_player.h` (Lua_Player)
- **Engine core:** `monsters.h` (monster_data, monster_definition, functions like accelerate_monster, damage_monster), `flood_map.h` (new_path, delete_path, pathfinding callbacks)
- **Boost:** `boost/bind.hpp` (function binding for dynamic limits)
- **Included inline:** `monster_definitions.h` (with DONT_REPEAT_DEFINITIONS guard)
