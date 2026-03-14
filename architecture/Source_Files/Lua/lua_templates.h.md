# Source_Files/Lua/lua_templates.h

## File Purpose
Provides C++ template classes for exposing C++ objects and enums to Lua via the Lua C API. These templates handle object registration, lifecycle management, getter/setter dispatch, and container access, enabling bidirectional C++ΓåöLua object interaction in the Aleph One game engine.

## Core Responsibilities
- **Object wrapping**: Create Lua userdata wrappers around indexed C++ objects with automatic lifetime management
- **Registry management**: Register metatables, method tables, and instance tables in the Lua registry
- **Method dispatch**: Route Lua `__index` / `__newindex` metamethods to registered getter/setter C functions
- **Validation**: Support pluggable validation functions to check if an object index is live
- **Enum support**: Extend base classes to support string mnemonics (names) for enum values with bidirectional lookup
- **Container iteration**: Provide Lua table-like access (indexing, iteration, length) to collections of objects
- **Custom fields**: Allow dynamic per-instance fields stored in a persistent custom-fields table

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `L_Class<name, index_t>` | template class | Base wrapper for indexed objects; registers metatable, get/set method tables, instance cache |
| `L_Enum<name, index_t>` | template class | Extends L_Class with enum mnemonic (string name) support and equality operator |
| `L_LazyEnum<name, index_t>` | template class | Specialization of L_Enum for error-tolerant mnemonic lookup (returns -1 or error on invalid input) |
| `L_Container<name, T>` | template class | Provides table-like iteration/indexing/length for a collection of type T objects |
| `L_EnumContainer<name, T>` | template class | Extends L_Container with string mnemonic lookup in addition to numeric indexing |
| `L_ObjectClass<name, object_t, index_t>` | template class | Holds both a numeric index and a generic object pointer; maintains object map for lifetime tracking |
| `always_valid` | struct | Functor; always returns true (no validation) |
| `ValidRange` | nested struct | Functor; validates index is in range [0, max) |
| `object_valid<...>` | template struct | Functor; checks if index exists in L_ObjectClass's `_objects` map |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `L_Class<...>::Valid` | `boost::function<bool(index_t)>` | static template member | Pluggable validation function; controls which indices are "alive" |
| `L_Container<...>::Length` | `boost::function<T::index_type(void)>` | static template member | Returns current container length for iteration and `__len` |
| `L_ObjectClass<...>::_objects` | `map<index_t, object_t>` | static template member | Stores object pointers by index; used by `Valid` and `Object()` accessors |

## Key Functions / Methods

### L_Class::Register
- **Signature**: `static void Register(lua_State *L, const luaL_reg get[] = 0, const luaL_reg set[] = 0, const luaL_reg metatable[] = 0)`
- **Purpose**: Initialize Lua metatable and method registry for this class; must be called once per class type before pushing instances
- **Inputs**: Lua state; optional arrays of getter/setter/metatable methods (luaL_reg pairs)
- **Outputs/Return**: None
- **Side effects**: Creates/registers metatable; creates method and instance tables in LUA_REGISTRYINDEX; registers global `is_<name>()` function
- **Calls**: `luaL_newmetatable`, `lua_setfield`, `lua_newtable`, `luaL_openlib`, `lua_settable`, `lua_setglobal`
- **Notes**: Getter methods are called as `get_func(object)` with result on stack; setter methods are `set_func(object, value)` with no return; custom fields (names starting with `_`) bypass method lookup and use a persistent custom-fields table

### L_Class::Push
- **Signature**: `static L_Class *Push(lua_State *L, index_t index)`
- **Purpose**: Push a Lua userdata wrapping the object at the given index onto the stack; reuses cached instance if already pushed
- **Inputs**: Lua state; object index
- **Outputs/Return**: Pointer to the L_Class userdata (or null if `Valid(index)` fails)
- **Side effects**: Pushes userdata onto stack; may create and cache a new instance; updates instance table in registry
- **Calls**: `Valid()`, `lua_newuserdata`, `luaL_getmetatable`, `lua_setmetatable`
- **Notes**: Instance caching prevents creating duplicate wrappers for the same index within a single session

### L_Class::Index
- **Signature**: `static index_t Index(lua_State *L, int index)`
- **Purpose**: Extract the wrapped object index from a Lua userdata at stack position `index`
- **Inputs**: Lua state; stack position
- **Outputs/Return**: Object index stored in the userdata
- **Side effects**: None; retrieves from userdata field
- **Calls**: None (direct cast)
- **Notes**: Raises `luaL_typerror` if userdata is not of this class type

### L_Class::Is
- **Signature**: `static bool Is(lua_State *L, int index)`
- **Purpose**: Check if the value at stack position `index` is a userdata of this class type (by metatable equality)
- **Inputs**: Lua state; stack position
- **Outputs/Return**: True if metatable matches, false otherwise
- **Side effects**: None (stack inspection only)
- **Calls**: `lua_touserdata`, `lua_getmetatable`, `lua_getfield`, `lua_rawequal`, `lua_pop`
- **Notes**: Safe to call on any stack value; returns false for non-userdata types

### L_Class::Invalidate
- **Signature**: `static void Invalidate(lua_State *L, index_t index)`
- **Purpose**: Remove an object from the instance cache and mark it as no longer live
- **Inputs**: Lua state; object index to invalidate
- **Outputs/Return**: None
- **Side effects**: Removes entry from instance table in registry; subsequent `Valid()` checks will fail
- **Calls**: `_push_instances_key`, `lua_gettable`, `lua_pushnumber`, `lua_pushnil`, `lua_settable`, `lua_pop`
- **Notes**: Called when C++ side destroys an object; any Lua references become stale (though userdata remains, `Valid()` will be false)

### L_Class::_get (metamethod)
- **Signature**: `static int _get(lua_State *L)` (Lua C function)
- **Purpose**: Implement Lua `__index` metamethod; dispatch property access to registered getter functions
- **Inputs**: Stack has `[1]=userdata, [2]=key (string)`
- **Outputs/Return**: Pushes result or nil; returns 1
- **Side effects**: Executes getter C function; may call Lua code (getter is `lua_pcall`'d)
- **Calls**: `lua_isstring`, `luaL_checktype`, `Valid`, `_push_custom_fields_table`, `lua_gettable`, `lua_istable`, `_push_get_methods_key`, `lua_pcall`
- **Notes**: Keys starting with `_` bypass method lookup and access custom fields table; non-callable keys return nil; errors in getter are wrapped with line info via `luaL_where`

### L_Class::_set (metamethod)
- **Signature**: `static int _set(lua_State *L)` (Lua C function)
- **Purpose**: Implement Lua `__newindex` metamethod; dispatch property assignment to registered setter functions
- **Inputs**: Stack has `[1]=userdata, [2]=key, [3]=value`
- **Outputs/Return**: Pushes nothing; returns 0
- **Side effects**: Executes setter C function or updates custom field; may call Lua code
- **Calls**: `luaL_checktype`, `lua_isstring`, `_push_custom_fields_table`, `_push_set_methods_key`, `lua_pcall`
- **Notes**: Keys starting with `_` create/update custom-field entries (lazy table creation); non-existent setter keys error with "no such index"

### L_Enum::ToIndex
- **Signature**: `static index_t ToIndex(lua_State *L, int index)`
- **Purpose**: Convert a Lua value (userdata, number, or string mnemonic) to an object index
- **Inputs**: Lua state; stack position
- **Outputs/Return**: Object index; on error, calls `luaL_error` (does not return)
- **Side effects**: May push/pop stack for mnemonic table lookup
- **Calls**: `_lookup`, `luaL_error`
- **Notes**: Provides three input modes: (1) wrapped userdata ΓåÆ unwrap index, (2) number ΓåÆ validate and use as index, (3) string ΓåÆ mnemonic lookup

### L_Enum::Register
- **Signature**: `static void Register(lua_State *L, const luaL_reg get[] = 0, const luaL_reg set[] = 0, const luaL_reg metatable[] = 0, const lang_def mnemonics[] = 0)`
- **Purpose**: Register enum class with mnemonic lookup; extends base L_Class registration
- **Inputs**: Lua state; get/set/metatable methods; array of `lang_def {name, value}` pairs (null-terminated)
- **Outputs/Return**: None
- **Side effects**: Calls parent Register; adds `__eq` and `__tostring` to metatable; loads mnemonic table into registry
- **Calls**: `L_Class::Register`, `luaL_getmetatable`, `lua_pushcfunction`, `lua_setfield`
- **Notes**: Enables `object == "mnemonic"` comparisons and automatic mnemonic display in `tostring()`

### L_Container::Register
- **Signature**: `static void Register(lua_State *L, const luaL_reg methods[] = 0, const luaL_reg metatable[] = 0)`
- **Purpose**: Register a container as a global table with Lua metatable for iteration and indexing
- **Inputs**: Lua state; optional container method functions and metatable methods
- **Outputs/Return**: None
- **Side effects**: Creates userdata and metatable; sets global with name; registers method table in registry
- **Calls**: `lua_newuserdata`, `luaL_newmetatable`, `lua_pushcfunction`, `lua_setfield`, `lua_setmetatable`, `lua_setglobal`
- **Notes**: Container is accessed in Lua as a table: `Name[index]` calls `_get`, `#Name` calls `_length`, `for x in Name() do...` iterates via `_iterator`

### L_Container::_get
- **Signature**: `static int _get(lua_State *L)` (Lua C function)
- **Purpose**: Implement `__index` metamethod for containers; map numeric index to T::Push or method lookup
- **Inputs**: Stack has `[1]=container userdata, [2]=key`
- **Outputs/Return**: Pushes object or method result; returns 1
- **Side effects**: May push object onto stack via T::Push
- **Calls**: `lua_isnumber`, `T::Valid`, `T::Push`, `_push_methods_key`, `lua_gettable`
- **Notes**: Numeric keys index the underlying collection; string keys look up container methods

### L_Container::_iterator
- **Signature**: `static int _iterator(lua_State *L)` (Lua C function)
- **Purpose**: Closure iterator for `for x in container() do...` loops
- **Inputs**: Upvalue at index 1 = current index; stack position irrelevant
- **Outputs/Return**: Pushes next object and incremented index; returns 1 (or pushes nil on end)
- **Side effects**: Modifies upvalue (current index) on each call
- **Calls**: `lua_tonumber`, `Length()`, `T::Valid`, `T::Push`, `lua_upvalueindex`, `lua_replace`
- **Notes**: Skips invalid indices; iteration stops when index >= Length()

### L_ObjectClass::Push
- **Signature**: `static L_ObjectClass *Push(lua_State *L, object_t object)`
- **Purpose**: Create a new userdata wrapper for a C++ object; allocate a new index and store in `_objects` map
- **Inputs**: Lua state; C++ object pointer/value
- **Outputs/Return**: Pointer to new userdata wrapper
- **Side effects**: Finds unused index; inserts into `_objects` map; creates and registers userdata on stack
- **Calls**: `lua_newuserdata`, `luaL_getmetatable`, `lua_setmetatable`
- **Notes**: Each call allocates a new wrapper; object lifetime is managed externally (map holds reference)

### L_ObjectClass::ObjectAtIndex
- **Signature**: `static object_t ObjectAtIndex(lua_State *L, index_t index)`
- **Purpose**: Retrieve C++ object from `_objects` map by index
- **Inputs**: Lua state (for error reporting); object index
- **Outputs/Return**: C++ object pointer/value
- **Side effects**: None
- **Calls**: `luaL_typerror` on missing index
- **Notes**: Used when index validity is already known or not critical

### L_ObjectClass::Object
- **Signature**: `static object_t Object(lua_State *L, int index)`
- **Purpose**: Extract C++ object from a Lua userdata at stack position `index`
- **Inputs**: Lua state; stack position
- **Outputs/Return**: C++ object from map (via Index ΓåÆ ObjectAtIndex)
- **Side effects**: None
- **Calls**: `L_Class::Index`, `ObjectAtIndex`
- **Notes**: Two-step unwrap: userdata ΓåÆ index ΓåÆ object pointer

## Control Flow Notes
**Initialization**: Engine calls `L_Class<T>::Register(L, ...)` once per C++ type at startup, populating metatables and method registries.

**Object dispatch**: When Lua script accesses `object.property`, the `__index` metamethod (`_get`) is invoked:
1. If key is a string starting with `_`, look in custom-fields table
2. Otherwise, look up getter in the registered method table and call it with the object as argument
3. Return result or nil

**Validation**: Each time an object is accessed, the pluggable `Valid(index)` function is checked (if not accessing "valid" property). Invalid objects log errors.

**Enum string lookup**: `L_Enum::_lookup` supports three forms: userdata ΓåÆ extract index, number ΓåÆ validate and use, string ΓåÆ search mnemonic table bidirectionally.

**Container iteration**: `for obj in Container() do ... end` works via upvalue-based closure; iterator skips invalid indices and stops at container length.

## External Dependencies
- **Lua C API** (`lua.h`, `lauxlib.h`, `lualib.h`): Stack manipulation, type checking, table/registry access, userdata creation
- **Boost** (`boost/function.hpp`): Function object wrapper for validation and length callbacks (allows both functor and function pointers)
- **STL** (`<sstream>`, `<map>`, `<string>`): String formatting, mnemonic lookups
- **Engine** (`lua_script.h`, `lua_mnemonics.h`): Game-specific callbacks, enum definitions; `L_Persistent_Table_Key()` function (defined elsewhere)
- **C series** (`cseries.h`): Platform abstraction, type definitions

**Note**: Actual getter/setter implementations, validation functors, and container types (T) are provided by code that instantiates these templates.
