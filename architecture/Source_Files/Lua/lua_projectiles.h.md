# Source_Files/Lua/lua_projectiles.h

## File Purpose
Header-only Lua binding declaration for projectile scripting support in Aleph One. Exposes individual projectile objects, projectile collections, and projectile type enumerations to the Lua VM using template-based C++ wrappers. Requires `HAVE_LUA` compile flag.

## Core Responsibilities
- Declare Lua class wrapper for individual projectile game objects
- Declare Lua container for iterating/accessing all live projectiles  
- Declare Lua enumeration type for projectile classification (type IDs)
- Declare Lua enum container for type lookup by name or numeric index
- Provide registration entry point to bind all projectile classes to Lua state
- Enable Lua scripts to query, iterate, and interact with game projectiles

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Projectile` | typedef (template class) | Lua binding wrapper for a single projectile object, indexed by numeric ID |
| `Lua_Projectiles` | typedef (template container) | Lua table-like collection of all projectile objects; supports iteration and indexed access |
| `Lua_ProjectileType` | typedef (template enum) | Lua enumeration for projectile type identifiers; supports equality comparison with numbers and string mnemonics |
| `Lua_ProjectileTypes` | typedef (template enum container) | Lua container for projectile type enum values; supports lookup by numeric ID or mnemonic string |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Projectile_Name` | `extern char[]` | static string | String literal "projectile"; template parameter naming the Lua class |
| `Lua_Projectiles_Name` | `extern char[]` | static string | String literal "Projectiles"; template parameter naming the Lua container |
| `Lua_ProjectileType_Name` | `extern char[]` | static string | String literal "projectile_type"; template parameter naming the Lua enum type |
| `Lua_ProjectileTypes_Name` | `extern char[]` | static string | String literal "ProjectileTypes"; template parameter naming the Lua enum container |

## Key Functions / Methods

### Lua_Projectiles_register
- **Signature:** `int Lua_Projectiles_register(lua_State *L)`
- **Purpose:** Register all projectile-related Lua classes and containers with the Lua VM; makes projectile objects scriptable from Lua code
- **Inputs:** `lua_State *L` ΓÇö active Lua state to register into
- **Outputs/Return:** `int` ΓÇö Lua API return code (typically 0 on success)
- **Side effects:** Modifies Lua registry; registers metatables, getter/setter methods, and global container objects
- **Calls:** (Implementation in `.cpp` file; header does not reveal callees)
- **Notes:** Entry point called during engine initialization; must be invoked before Lua scripts interact with projectiles

## Control Flow Notes
This is a declaration-only header. It sets up the type bindings but does not implement initialization logic. Called early in the Lua subsystem setup (likely during script engine init) before any Lua script attempts to access projectile objects. The template instantiations delegate to `L_Class`, `L_Container`, and `L_Enum` template implementations (from `lua_templates.h`) which handle Lua stack manipulation, metatables, and method dispatch.

## External Dependencies
- **Lua 5.1 API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö core Lua C interface and auxiliary library functions  
- **Template implementations:** `lua_templates.h` ΓÇö defines `L_Class`, `L_Container`, `L_Enum`, `L_EnumContainer` template classes for binding C++ objects to Lua  
- **Core headers:** `cseries.h` (cross-platform utilities), `config.h` (build configuration)  
- **Defined elsewhere:** actual string values `Lua_Projectile_Name`, `Lua_Projectiles_Name`, `Lua_ProjectileType_Name`, `Lua_ProjectileTypes_Name` (implementation file); registration logic in corresponding `.cpp` file
