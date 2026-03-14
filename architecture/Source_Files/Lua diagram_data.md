# Source_Files/Lua/language_definition.h
## File Purpose
Lua language binding definitions that map human-readable symbolic names to numeric constants for game entities. Provides dual naming conventions (underscore-prefixed and plain) for backwards compatibility. Used to allow Lua scripts to reference items, monsters, sounds, and game mechanics by name rather than raw hex values.

## Core Responsibilities
- Define symbolic mnemonics for inventory items (weapons, magazines, powerups)
- Define symbolic mnemonics for enemy/NPC types and variants
- Define symbolic mnemonics for damage sources
- Define symbolic mnemonics for monster classifications and behavioral states
- Define symbolic mnemonics for player input actions and monster actions
- Define symbolic mnemonics for visual effects (fades and screen tints)
- Define symbolic mnemonics for sound effect IDs
- Define symbolic mnemonics for projectile/weapon fire types
- Define symbolic mnemonics for polygon/terrain properties and triggers
- Define symbolic mnemonics for game modes (CTF, KOTH, etc.)
- Provide backwards-compatible dual naming (with/without `_` prefix)

## External Dependencies
- `#include "config.h"` ΓÇö provides `HAVE_LUA` preprocessor conditional
- References undefined symbolic constants (defined elsewhere):
  - `_game_of_most_points`, `_game_of_most_time`, etc. (game scoring modes)
  - `_panel_is_oxygen_refuel`, `_panel_is_shield_refuel`, etc. (panel types)
  - `_weapon_fist`, `_weapon_pistol`, etc. (weapon identifiers)


# Source_Files/Lua/lapi.h
## File Purpose
Declares auxiliary functions for the Lua C API. This minimal header provides internal helper functions for Lua object manipulation at the C level, specifically for pushing tagged values onto the Lua stack.

## Core Responsibilities
- Declare internal Lua API helper functions
- Provide stack manipulation interface for TValue objects
- Support internal Lua state operations during script execution

## External Dependencies
- `config.h`: Conditional compilation guard (`HAVE_LUA`)
- `lobject.h`: Provides `TValue` definition and Lua object type system (tags, GCObject union, type checking macros)
- Implicit: `lua.h` (referenced in copyright notice; provides `lua_State`)

# Source_Files/Lua/lauxlib.h
## File Purpose

Lua auxiliary library header providing convenience functions and macros for building Lua libraries and extending Lua with C code. Defines the public API for type checking, argument validation, library registration, string buffering, and reference management.

## Core Responsibilities

- Define the `luaL_Reg` struct for registering arrays of C functions as Lua libraries
- Provide type-checking and argument-validation helper functions (luaL_check*, luaL_opt*)
- Supply convenience macros for common operations (type tests, conversions, option parsing)
- Implement growable buffer mechanism (`luaL_Buffer`) for efficient string construction
- Define library registration functions (`luaL_register`, `luaI_openlib`)
- Offer error handling utilities with function argument context
- Manage Lua registry references (`luaL_ref`/`luaL_unref`) to prevent garbage collection
- Support code loading from files, buffers, and strings without immediate execution
- Provide backward compatibility for multiple Lua versions

## External Dependencies

- `lua.h`: Core Lua C API (defines `lua_State`, `lua_CFunction`, `LUA_REGISTRYINDEX`, `LUA_MULTRET`, error codes)
- `config.h`: Build configuration flag `HAVE_LUA`
- Standard C: `<stddef.h>` (size_t), `<stdio.h>` (FILE)

# Source_Files/Lua/lcode.h
## File Purpose
Header for Lua's bytecode code generator. Declares the interface for emitting VM instructions, managing expression compilation, and handling code generation during the parsing phase. Part of Lua's compilation pipeline that converts parsed AST into executable bytecode.

## Core Responsibilities
- Emit bytecode instructions (ABC and ABx formats) to function prototypes
- Convert expressions to registers or constants (expression coercion)
- Manage the constant table (strings, numbers) and assign constant indices
- Generate control flow instructions (jumps, conditional branches, returns)
- Patch jump addresses for forward/backward control flow
- Handle unary and binary operator code generation
- Reserve and track register allocation during compilation
- Set line number information for debugging

## External Dependencies
- **llex.h**: Token types and lexical state (LexState)
- **lobject.h**: Lua value types (TValue, TString, Proto, FuncState)
- **lopcodes.h**: VM opcode definitions (OpCode enum) and instruction encoding macros
- **lparser.h**: Expression descriptor (expdesc), function state (FuncState), upvalue descriptors

# Source_Files/Lua/ldebug.h
## File Purpose
Header for Lua's Debug Interface auxiliary functions module. Provides macro utilities and declares error reporting and code validation functions used throughout the Lua runtime to handle errors and debug hooks.

## Core Responsibilities
- Define debugging macros for bytecode operations (program counter relative offset, line info lookup, hook count reset)
- Declare type-specific error reporting functions (type errors, concatenation, arithmetic, ordering)
- Declare general runtime error and error message dispatch functions
- Declare code and instruction validation functions for bytecode integrity

## External Dependencies
- **Includes**: `config.h`, `lstate.h`
- **Types used** (defined elsewhere): `lua_State`, `TValue`, `StkId`, `Proto`, `Instruction`
- **LUAI_FUNC macro**: Marks these as Lua internal C API functions

# Source_Files/Lua/ldo.h
## File Purpose
Header for Lua's stack and call management system. Defines the interface for function call execution, protected execution contexts, and stack/CallInfo array allocation. Central to the VM's call semantics.

## Core Responsibilities
- Stack safety macros (`luaD_checkstack`, `incr_top`) for bounds checking and growth
- Stack pointer serialization/restoration (`savestack`/`restorestack`) for snapshots across reallocation
- CallInfo array index serialization (`saveci`/`restoreci`) for position tracking
- Function call protocol (precall setup, call execution, postcall cleanup)
- Protected execution (`luaD_pcall`, `luaD_rawrunprotected`) for exception handling
- Stack and CallInfo array reallocation and growth
- Parser invocation with error recovery
- Hook invocation for debugger/profiler integration
- Error propagation (`luaD_throw`, `luaD_seterrorobj`)

## External Dependencies
- **Includes**: `lobject.h` (TValue, Closure, Proto), `lstate.h` (lua_State, CallInfo, global_State), `lzio.h` (ZIO, input streams)
- **Defined elsewhere**: `lua_State` structure, `TValue` union, `StkId` (alias for `TValue *`), `CallInfo` struct, `Instruction` (bytecode), `lua_Reader` callback

# Source_Files/Lua/lfunc.h
## File Purpose
Header file declaring auxiliary functions for manipulating Lua function prototypes and closures. Provides memory allocation, upvalue management, and cleanup routines for Lua's closure system.

## Core Responsibilities
- Factory functions for creating Lua and C closures
- Prototype (function template) creation and destruction
- Upvalue (captured variable) creation, lookup, and lifecycle management
- Memory deallocation for prototypes, closures, and upvalues
- Local variable name lookup by program counter

## External Dependencies
- `config.h` ΓÇô Build configuration (e.g., `HAVE_LUA` guard)
- `lobject.h` ΓÇô Type definitions: `Proto`, `Closure`, `UpVal`, `Table`, `TValue`, `StkId`, `lua_State`
- Lua C API: `lua_CFunction` (C function pointer type)

# Source_Files/Lua/lgc.h
## File Purpose
Header file for Lua's garbage collector implementation. Defines the tri-color marking scheme, GC states, write barrier macros for maintaining GC invariants, bit manipulation utilities, and function declarations for GC operations like incremental collection and finalization.

## Core Responsibilities
- Define garbage collector states (pause, propagate, sweep, finalize)
- Provide tri-color marking macros (white/black/gray) for incremental GC
- Implement write barrier macros to maintain GC invariants when references change
- Define bit manipulation macros for marking object metadata
- Declare GC entry points and helper functions
- Define GC threshold checking mechanism

## External Dependencies
- `config.h` ΓÇö project configuration; checked for `HAVE_LUA`
- `lobject.h` ΓÇö defines `GCObject`, `GCheader`, `Table`, `UpVal`, `lua_State`, `lu_byte` (marked field, type tag, etc.)
- Symbols defined elsewhere: `luaD_reallocstack`, `G(L)` (global state accessor), `iscollectable`, `gcvalue`, `obj2gco`

# Source_Files/Lua/llex.h
## File Purpose
Header for Lua's lexical analyzer. Defines token types, semantic information structures, lexer state, and the public API for tokenization of Lua source code.

## Core Responsibilities
- Define reserved word tokens and operator token constants (enum RESERVED)
- Define Token and SemInfo structures for representing parsed tokens and their values
- Define LexState structure maintaining lexer state during tokenization
- Declare lexer initialization, input setup, and token manipulation functions
- Provide error reporting utilities for lexical/syntax errors

## External Dependencies
- `config.h` ΓÇö Compilation config; guards module with `HAVE_LUA`
- `lobject.h` ΓÇö Type definitions: `TString`, `lua_Number`
- `lzio.h` ΓÇö Buffered input: `ZIO` (stream), `Mbuffer` (token buffer)
- `lua_State` ΓÇö Lua runtime (opaque; defined elsewhere)
- `FuncState` ΓÇö Parser state (opaque to lexer; stored as pointer in LexState)
- Macros `LUAI_FUNC`, `LUAI_DATA` ΓÇö Function/data visibility annotations

# Source_Files/Lua/llimits.h
## File Purpose
This header defines Lua VM internal limits, platform-specific type aliases, and utility macros for the Lua 5.1 runtime. It establishes maximum constraints for stack depth, memory allocation, and string tables, plus helper macros for type casting, assertions, and thread synchronization used throughout the Lua engine.

## Core Responsibilities
- Define portable integer types (lu_int32, lu_mem, l_mem, lu_byte) abstracting platform differences
- Establish safe upper bounds for stack (MAXSTACK=250), memory (MAX_LUMEM, MAX_INT), and string tables
- Provide casting macros (cast, cast_byte, cast_num) and assertion helpers for type safety and debugging
- Define threading primitives (lua_lock/lua_unlock) for optional synchronization
- Declare Instruction type for VM bytecode
- Suppress compiler warnings via UNUSED() macro for unused parameters

## External Dependencies
- `<limits.h>` ΓÇö C standard integer limits (INT_MAX)
- `<stddef.h>` ΓÇö Standard types (size_t, NULL)
- `lua.h` ΓÇö Public Lua API (transitively pulls luaconf.h for LUAI_* macros)
- **Symbols expected from luaconf.h (defined elsewhere):** LUAI_UINT32, LUAI_UMEM, LUAI_MEM, LUAI_USER_ALIGNMENT_T, LUAI_UACNUMBER, optional lua_assert override
- **config.h guard:** Conditional compilation (HAVE_LUA) for Aleph One integration

# Source_Files/Lua/lmem.h
## File Purpose
Header file providing Lua's memory management interface. Defines macros for safe allocation, reallocation, and deallocation of memory blocks and arrays, with overflow checking and error handling.

## Core Responsibilities
- Declare core memory allocation functions (`luaM_realloc_`, `luaM_toobig`, `luaM_growaux_`)
- Provide convenience macros for single-object allocation (`luaM_new`, `luaM_malloc`)
- Provide vector/array allocation macros (`luaM_newvector`, `luaM_reallocvector`, `luaM_growvector`)
- Implement overflow-safe reallocation with size checking (`luaM_reallocv`)
- Supply standardized error message constant

## External Dependencies
- `config.h`: Conditional `HAVE_LUA` guard
- `llimits.h`: `MAX_SIZET`, `cast()` macro
- `lua.h`: `lua_State` opaque type
- Implementation file: `lmem.c` (not shown)

# Source_Files/Lua/lobject.h
## File Purpose
Defines internal memory representation and type system for Lua 5.1 runtime objects. Declares tagged value structures (TValue), garbage-collected types, and accessor/setter macros for the virtual machine to manipulate objects uniformly.

## Core Responsibilities
- Define type tags and distinguish collectable vs non-collectable values
- Provide TValue (tagged value) union structure combining value data + type tag
- Define GCObject union encapsulating all garbage-collected types
- Declare String (TString), Userdata (Udata), Function Prototype (Proto), Closure, and Table structures
- Provide type-checking macros (ttisnil, ttisnumber, etc.)
- Provide value accessor macros (ttype, gcvalue, nvalue, clvalue, hvalue, etc.)
- Provide value setter macros (setnilvalue, setnvalue, setsvalue, etc.) with liveness checks
- Define table hashing and size computation utilities

## External Dependencies
- **config.h**: HAVE_LUA conditional guard
- **llimits.h**: lu_byte, lu_int32, lu_mem, l_mem, L_Umaxalign, Instruction, cast, check_exp, lua_assert macros
- **lua.h**: type tag constants (LUA_TNIL, LUA_TNUMBER, etc.), lua_Number, lua_State, lua_CFunction types, LUA_IDSIZE constant
- **stdarg.h**: va_list for variadic functions

# Source_Files/Lua/lopcodes.h
## File Purpose

Defines the Lua virtual machine instruction format, opcodes, and helper macros for encoding/decoding instructions. This header specifies how VM instructions are laid out in memory and provides compile-time macros to manipulate instruction fields.

## Core Responsibilities

- Define instruction bit layout (opcode, A, B, C arguments and their positions)
- Enumerate all Lua VM opcodes (OP_MOVE, OP_CALL, OP_RETURN, etc.)
- Provide macros to extract/insert instruction fields (GET_OPCODE, GETARG_A, etc.)
- Define instruction creation macros (CREATE_ABC, CREATE_ABx)
- Manage RK indices (constant/register disambiguation via BITRK flag)
- Define opcode argument modes and property queries
- Calculate argument limits (MAXARG_A, MAXARG_B, MAXARG_C, MAXARG_Bx, MAXARG_sBx)

## External Dependencies

- **config.h** ΓÇô `HAVE_LUA` guard
- **llimits.h** ΓÇô `Instruction` typedef, `MAX_INT`, `cast()` macro, `lu_byte`, `lu_int32`
- **lua.h** (included transitively) ΓÇô Lua API definitions

# Source_Files/Lua/lparser.h
## File Purpose
Header file for Lua's parser and code generator. Defines expression descriptors, function compilation state, and upvalue tracking structures used during parsing and bytecode generation.

## Core Responsibilities
- Define expression kind enum (`expkind`) categorizing all Lua expression types
- Define expression descriptor (`expdesc`) with value encoding and control-flow patch lists
- Define upvalue metadata structure for closure variable handling
- Define function state (`FuncState`) tracking compiler context during code generation
- Declare parser entry point `luaY_parser`

## External Dependencies
- **config.h**: Build configuration (HAVE_LUA guard)
- **llimits.h**: Type definitions (lu_byte, lu_mem, Instruction, LUAI_* constants)
- **lobject.h**: Lua runtime objects (Proto, Table, TValue, GCObject)
- **lzio.h**: Buffered I/O (ZIO, Mbuffer, lua_Reader callback type)

# Source_Files/Lua/lstate.h
## File Purpose
Core Lua 5.1 VM state header defining the runtime representation of global and per-thread execution states. Manages call stack metadata, garbage collection state, string interning, memory allocation, and thread management for the embedded Lua runtime in Aleph One.

## Core Responsibilities
- Define `global_State` struct: shared state across all Lua threads (memory allocator, GC lists, string table, metatables, registry)
- Define `lua_State` struct: per-thread execution state (stack, call info, error handling, hooks, environment)
- Define `stringtable` for string interning with collision chains
- Define `CallInfo` for call stack frame metadata
- Define `GCObject` union: all garbage-collectable types (strings, tables, functions, threads, etc.)
- Provide access macros (`gt()`, `G()`, `registry()`) and type-checking/conversion macros
- Declare thread creation/destruction entry points

## External Dependencies
- `lua.h` ΓÇô Lua public API types and constants (`lua_State`, `LUA_*` type tags)
- `lobject.h` ΓÇô Lua value types (`TValue`, `GCObject`, `TString`, `Table`, `Closure`, `Proto`, `UpVal`)
- `ltm.h` ΓÇô Tag method enumeration and metatable lookups
- `lzio.h` ΓÇô Buffered I/O structures (`Mbuffer` for chunk loading)
- `ldo.c` ΓÇô Implementation of `luaE_newthread`, `luaE_freethread`, and `lua_longjmp` definition
- Type references: `Instruction` (VM opcode), `lua_Alloc` (memory allocator), `lua_Hook` (debug hook), `lua_CFunction` (C function callback)

# Source_Files/Lua/lstring.h
## File Purpose
Header file for Lua's string table (interned string pool). Defines the public interface for creating and managing strings and userdata, including sizing macros and string creation convenience wrappers. All strings created via Lua are stored in a hash table managed by this module.

## Core Responsibilities
- Define macros to calculate memory size of strings (`TString`) and userdata (`Udata`)
- Provide convenience macros for string creation from C strings and string literals
- Declare interface for resizing the string table and allocating new userdata
- Mark strings as "fixed" (exempt from garbage collection)

## External Dependencies
- `config.h` ΓÇô Platform configuration.
- `lgc.h` ΓÇô Garbage collector macros: `l_setbit`, `FIXEDBIT` bit manipulation.
- `lobject.h` ΓÇô Type definitions: `TString`, `Udata`, `GCObject`.
- `lstate.h` ΓÇô Lua state type: `lua_State`.

# Source_Files/Lua/ltable.h
## File Purpose
Header defining the public C interface for Lua's hash table (associative array) implementation. Declares functions for table creation, lookup, assignment, resizing, and iteration. Part of the Lua runtime embedded in the AlephOne game engine.

## Core Responsibilities
- Define accessor macros (`gnode`, `gkey`, `gval`, `gnext`, `key2tval`) for internal table node structures
- Declare typed lookup/assignment functions (by integer key, string key, or generic TValue)
- Declare table lifecycle operations (creation, resizing, freeing)
- Declare table iteration and length queries
- Expose debug introspection functions (conditional on `LUA_DEBUG`)

## External Dependencies
- `config.h` ΓÇö Provides `HAVE_LUA` guard; configuration flags (AlephOne-specific)
- `lobject.h` ΓÇö Defines `Table`, `Node`, `TValue`, `TKey`, `TString`, `StkId` types and TValue/GC macros
- `LUAI_FUNC` macro ΓÇö Controls function visibility/export (likely `extern` or `__declspec`)
- Lua state type `lua_State` ΓÇö Defined elsewhere (memory allocator context)

# Source_Files/Lua/ltm.h
## File Purpose
Header file defining Lua's tag method (metamethod) system, enabling operator overloading and object behavior customization through metatables. Declares the tag method type enumeration, provides fast-path lookup macros with caching optimization, and exports core initialization functions.

## Core Responsibilities
- Enumerate all tag method types (TM_INDEX, TM_ADD, TM_CONCAT, etc.) with strict ordering
- Provide fast-path lookup macros exploiting metatable flags for known-absent methods
- Declare tag method retrieval functions for table-based and object-based access
- Initialize the global tag method name strings during Lua startup
- Reference interned tag method names used throughout the VM

## External Dependencies
- `#include "lobject.h"` ΓÇö TValue, Table, TString, GCObject types
- `#include "config.h"` ΓÇö Preprocessor conditionals
- Implicit: `G(l)` macro (extracts global state from lua_State, defined elsewhere)
- Implicit: tag method flags bit layout assumption in metatable


# Source_Files/Lua/lua.h
## File Purpose

Public C API header for Lua 5.1.2, an extensible extension language. Defines the interface for embedding Lua in C/C++ applications, including state management, stack operations, type system, and execution control.

## Core Responsibilities

- **State lifecycle**: Create, manage, and destroy Lua VM instances
- **Stack API**: Push/pop values, manipulate stack positions, type introspection
- **Value conversion**: Bridge between C native types and Lua types
- **Execution**: Load and execute Lua code, call functions, handle coroutines
- **Memory management**: Custom allocator support, garbage collection control
- **Debug API**: Stack introspection, hook installation, breakpoint support
- **Type system**: Define 8 core types (nil, boolean, number, string, table, function, userdata, thread)
- **C function callbacks**: Register native C functions callable from Lua

## External Dependencies

- **config.h**: Conditional compilation (e.g., `HAVE_LUA`)
- **luaconf.h**: Platform-specific configuration (type sizes, allocators, dynamic linking)
- **Standard C**: `<stdarg.h>`, `<stddef.h>` (variadic args, size_t)
- **Internal symbols** (defined elsewhere): All function implementations, `lua_State` struct definition, `LUA_API` macro (usually `extern` or DLL visibility)

---

**Notes:**
- All exported functions are marked `LUA_API` (defined in luaconf.h; typically `extern` or `__declspec(dllexport/dllimport)`)
- Stack indices: positive = absolute (1-based), negative = relative to top (ΓêÆ1 = top)
- Pseudo-indices (< 0) access VM globals/registry without consuming stack: `LUA_REGISTRYINDEX`, `LUA_GLOBALSINDEX`, etc.
- Macro helpers like `lua_pop()`, `lua_newtable()`, `lua_register()` simplify common patterns

# Source_Files/Lua/lua_hud_objects.cpp
## File Purpose
Implements Lua bindings for HUD (heads-up display) objects and engine globals, exposing graphical rendering (images, shapes, fonts), screen properties (dimensions, clip regions, field of view), and game state (players, difficulty, scoring) to Lua scripts. Supports creation, manipulation, and drawing of HUD primitives.

## Core Responsibilities
- Register Lua enum types and container classes for game concepts (item types, player colors, game types, difficulty levels, renderer types, texture types, fade effects)
- Provide Lua wrappers for blitter classes (`Image_Blitter`, `Shape_Blitter`) with property getters/setters (dimensions, tint, rotation, crop rectangles)
- Expose screen/view properties to Lua (window dimensions, clip rect, world rect, map rect, terminal rect, field of view, renderer type, UI scale preferences)
- Implement player and game query APIs (player list, kills, ranking, color, team, difficulty, time remaining, scoring mode, game ticks, version)
- Handle HUD drawing commands (fill/frame rectangles, draw images/shapes/text) via `Lua_HUDInstance()`
- Manage color lookups supporting both named (r/g/b/a) and indexed (1/2/3/4) color channel access
- Register lighting fader queries (liquid/damage effects with type, color, active state)

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` (core interpreter and auxiliary library functions)
- **Game engine:** `items.h`, `player.h`, `motion_sensor.h`, `screen.h`, `shell.h`, `network.h`, `render.h` (game world data)
- **Rendering:** `HUDRenderer.h`, `HUDRenderer_Lua.h`, `Image_Blitter.h`, `OGL_Blitter.h`, `Shape_Blitter.h`, `FontHandler.h` (blitters and HUD rendering)
- **Effects/lighting:** `fades.h`, `OGL_Faders.h` (visual effects system)
- **Resources:** `collection_definition.h`, `alephversion.h` (asset management)
- **Templates:** `lua_templates.h`, `lua_objects.h`, `lua_map.h` (Lua binding framework and other domain bindings)
- **Boost:** `boost/bind.hpp`, `boost/shared_ptr.hpp` (utilities, though `shared_ptr` not used in this file)

# Source_Files/Lua/lua_hud_objects.h
## File Purpose
Declares Lua C bindings for HUD (Heads-Up Display) objects and enums in the Aleph One game engine. Provides the glue layer allowing Lua scripts to interact with player, game, and screen HUD state via templated Lua class/enum wrappers.

## Core Responsibilities
- Define Lua class proxies (`Lua_HUDPlayer`, `Lua_HUDGame`, `Lua_HUDScreen`) for HUD game objects
- Define Lua enum proxies (`Lua_InventorySection`, `Lua_RendererType`, `Lua_SensorBlipType`, `Lua_TextureType`) for game enums
- Define Lua container types (`Lua_InventorySections`, `Lua_RendererTypes`, `Lua_SensorBlipTypes`, `Lua_TextureTypes`) for enum collections
- Expose a registration function to bind all these types to a Lua runtime
- Support Lua scripts reading/writing HUD state through the Lua stack

## External Dependencies

- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö Lua 5.1 stack manipulation, type checking, metatable registration
- **Template library:** `lua_templates.h` ΓÇö `L_Class<>`, `L_Enum<>`, `L_EnumContainer<>` template class definitions for Lua bindings
- **Game headers:** `items.h`, `map.h` ΓÇö Likely provide game object definitions referenced by HUD wrappers (defined elsewhere)
- **Platform layer:** `cseries.h` ΓÇö Cross-platform macros and types (SDL, endianness, etc.)
- **Config:** `config.h` ΓÇö Feature flags (e.g., `HAVE_LUA`, `HAVE_OPENGL`)

# Source_Files/Lua/lua_hud_script.cpp
## File Purpose
Implements Lua scripting support for the HUD system in Aleph One. Manages a Lua state for loading and executing HUD scripts, handling lifecycle callbacks (init, draw, resize, cleanup), and tracking game resource collections referenced by scripts. Provides conditional stubs when Lua support is unavailable.

## Core Responsibilities
- Create and manage a Lua VM instance via the `LuaHUDState` singleton
- Load Lua script buffers and execute them in the game context
- Invoke Lua trigger functions at key HUD lifecycle points (initialization, frame rendering, window resize, shutdown)
- Register C++ functions for Lua scripts to call (e.g., HUD object manipulation)
- Parse Lua script declarations of resource collections needed for loading
- Provide public C-linkage API functions for engine integration
- Load HUD scripts from user preferences during game startup

## External Dependencies
- **Lua 5.1.2 libraries:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö VM, auxiliary functions, standard library bindings
- **Boost:** `shared_ptr` (RAII wrapper for lua_State), `iostreams` (unused in this file but included)
- **Game engine:** `preferences.h` (environment_preferences), `interface.h` (FileSpecifier, mark_collection_for_loading/unloading, game error state), `mouse.h`, `Logging.h` (logWarning)
- **HUD bindings:** `lua_hud_objects.h` ΓåÆ `Lua_HUDObjects_register()` (defined elsewhere)
- **Externs defined elsewhere:** `AngleConvert`, `MotionSensorActive`, `world_view`, `static_world`, `L_Error()`, `L_Persistent_Table_Key()`

# Source_Files/Lua/lua_hud_script.h
## File Purpose
Header file declaring the Lua HUD scripting interface for Aleph One. Provides callback entry points for Lua scripts to hook into the game's HUD lifecycle (init, cleanup, draw, resize) and functions to load/unload Lua HUD scripts.

## Core Responsibilities
- Declare callback functions triggered by the HUD system (init, cleanup, draw, resize)
- Provide script loading/execution interface (`LoadLuaHUDScript`, `RunLuaHUDScript`)
- Manage Lua HUD script lifecycle (load, run, close)
- Support memory management for Lua collections during loading
- Bridge C/C++ game engine with Lua HUD scripts

## External Dependencies
- `config.h` ΓÇô conditional compilation (`HAVE_LUA` guard)
- `cseries.h` ΓÇô base type definitions and macros (included indirectly)
- Lua library (linked externally; not included here)

# Source_Files/Lua/lua_map.cpp
## File Purpose
Implements Lua bindings for map-related game objects in the Aleph One game engine. Exposes collections, polygons, sides, lines, endpoints, platforms, lights, tags, media, annotations, fog, and level metadata to Lua scripts through getter/setter methods and container classes.

## Core Responsibilities
- Register Lua classes and containers for map geometry (polygons, lines, endpoints, sides)
- Implement property accessors (getters/setters) for map entity attributes (heights, textures, states)
- Expose dynamic world state (platforms, lights, tags, media) to Lua scripting
- Provide compatibility wrappers for old Lua scripts via embedded Lua code
- Bridge C++ map data structures with Lua's type system using template-based wrappers

## External Dependencies
- **Lua headers**: `lua.h`, `lauxlib.h`, `lualib.h` (C Lua API)
- **Lua template framework**: `lua_templates.h` (defines `L_Class<>`, `L_Enum<>`, `L_Container<>`)
- **Map geometry**: `map.h` (polygon, line, endpoint, side data structures)
- **Dynamic data**: `lightsource.h`, `media.h`, `platforms.h` (dynamic world state)
- **Support headers**: `lua_monsters.h`, `lua_objects.h` (cross-subsystem Lua bindings)
- **Engine systems**: `OGL_Setup.h` (fog rendering), `SoundManager.h` (audio control)
- **Boost**: `boost/bind.hpp` (function binding, used in comments but not actively in this file)
- **Defined elsewhere**: `get_collection_definition()`, `get_panel_class()`, `get_control_panel_definition()`, `get_endpoint_data()`, `get_line_data()`, `get_platform_data()`, `get_polygon_data()`, `get_light_data()`, `get_media_data()`, `set_platform_state()`, `adjust_platform_*()`, `set_light_status()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `assume_correct_switch_position()`, `calculate_level_completion_state()`, `OGL_GetFogData()`, `number_of_terminal_texts()` (all accessed from external modules)

# Source_Files/Lua/lua_map.h
## File Purpose
Declares Lua bindings for game world map structures in Aleph One. Provides wrapper classes that expose C++ map geometry, lighting, and object data to Lua scripting, enabling level designers and scripters to manipulate map elements dynamically.

## Core Responsibilities
- Declare `L_Class`, `L_Container`, and `L_Enum` template instantiations for map-related types
- Export extern collection names used as template parameters for Lua binding registration
- Define typedef pairs combining template base classes with collection-specific names
- Declare the registration function that wires all map bindings into a Lua state

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô Core Lua embedding interfaces.
- **Engine templates:** `lua_templates.h` ΓÇô Template definitions for `L_Class`, `L_Container`, `L_Enum`, `L_EnumContainer`.
- **Game world structs:** `map.h` ΓÇô Core map data structures (polygons, lines, sides, objects, etc.).
- **Lighting system:** `lightsource.h` ΓÇô Light source definitions and accessors.
- **Build config:** `config.h` ΓÇô Conditional compilation; `HAVE_LUA` gate.
- **C++ series library:** `cseries.h` ΓÇô Common types and macros.

# Source_Files/Lua/lua_mnemonics.h
## File Purpose
Defines string-to-integer mnemonic lookup tables for Lua scripting. Maps human-readable game element names (e.g., "knife", "rocket explosion", "kill monsters") to numeric constants used internally by the engine.

## Core Responsibilities
- Provide mnemonic mappings for game collections (creatures, items, weapons)
- Define enumerations for game states (completion, difficulty, monster actions)
- Supply effect, sound, and visual rendering parameters as named constants
- Enable Lua scripts to reference game entities by readable names instead of magic numbers
- Support multiple game systems: combat, inventory, level geometry, audio, visual effects

## External Dependencies
- **`config.h`** ΓÇô Provides configuration symbols (e.g., `HAVE_LUA`)
- **`int32` type** ΓÇô Assumed defined in a broader headers (likely `<stdint.h>` or equivalent)
- **No function calls or dynamic dependencies** within this file

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

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` (conditional on `HAVE_LUA`)
- **Templates:** `lua_templates.h` (L_Class, L_Enum, L_Container, L_EnumContainer)
- **Sibling Lua bindings:** `lua_map.h` (Lua_Polygon, Lua_DamageType), `lua_objects.h` (Lua_EffectType, Lua_Sound, Lua_ItemType), `lua_player.h` (Lua_Player)
- **Engine core:** `monsters.h` (monster_data, monster_definition, functions like accelerate_monster, damage_monster), `flood_map.h` (new_path, delete_path, pathfinding callbacks)
- **Boost:** `boost/bind.hpp` (function binding for dynamic limits)
- **Included inline:** `monster_definitions.h` (with DONT_REPEAT_DEFINITIONS guard)

# Source_Files/Lua/lua_monsters.h
## File Purpose
Declares Lua C bindings for exposing the game's monster system to Lua scripts. Provides typed wrappers for individual monsters, monster collections, and monster action states using template-based classes that bridge C++ data structures with Lua.

## Core Responsibilities
- Define Lua class and container types for monsters and monster actions
- Declare string mnemonics for Lua class names ("monster", "Monsters", "monster_action")
- Register a function to initialize all monster-related Lua bindings with a Lua state
- Support type-safe access to monster data from Lua scripts using the template system from `lua_templates.h`

## External Dependencies
- **Lua C API headers:** `lua.h`, `lauxlib.h`, `lualib.h` (5.1)
- **Game engine headers:** `config.h`, `cseries.h`, `map.h`, `monsters.h` (for underlying data structures)
- **Template utilities:** `lua_templates.h` (defines `L_Class<>`, `L_Container<>`, `L_Enum<>` base templates)
- **Extern symbols used but not defined here:** Monster data structures and constants from `monsters.h` and `map.h`

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

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h` (Lua 5.0ΓÇô5.1 API)
- **Game world:** `effects.h`, `items.h`, `monsters.h`, `scenery.h`, `player.h`, `scenery_definitions.h` (object definitions and management)
- **Sound system:** `SoundManager.h` (custom sound registration)
- **Lua engine bindings:** `lua_map.h`, `lua_templates.h` (template base classes and polygon references)
- **Build system:** `config.h` (conditional compilation HAVE_LUA)
- **Boost:** `boost/bind.hpp` (function binding for dynamic limit callbacks)
- **Defined elsewhere:** `get_object_data()`, `remove_map_object()`, `new_effect()`, `new_item()`, `new_scenery()`, `play_object_sound()`, `damage_scenery()`, `deanimate_scenery()`, `randomize_scenery_shape()`, `get_dynamic_limit()`, `add_object_to_polygon_object_list()`, `remove_object_from_polygon_object_list()`

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

## External Dependencies

- **Lua C API** (lua.h, lauxlib.h, lualib.h): Lua 5.1 core interpreter; stack operations, metatables, type checking
- **lua_templates.h**: Template base classes (L_Class, L_Enum, L_Container, L_LazyEnum, L_EnumContainer) that implement the Lua/C++ bridge mechanism
- **items.h**: Game item type definitions and prototypes (for Lua_Item context)
- **map.h**: Game world structures and object management (for Lua_Effect, Lua_Scenery context)
- **cseries.h**: Cross-platform compatibility and standard utilities
- **config.h**: Compile-time configuration (HAVE_LUA feature flag)

# Source_Files/Lua/lua_player.cpp
## File Purpose
Implements Lua scripting bindings for player-related game mechanics in Marathon: Aleph One. Exposes player state, game settings, and engine functionality to Lua scripts through a comprehensive C API bridge, including player manipulation, camera control, weapons, inventory, HUD overlays, and music management.

## Core Responsibilities
- Register all player-related Lua classes and methods with the Lua VM
- Provide getters/setters for player properties (position, energy, weapons, team, etc.)
- Implement camera system with path point and angle animation
- Manage action flags (input state) for player control
- Handle inventory and weapon selection
- Control HUD overlays and crosshairs state
- Expose compass/beacon system for navigation aids
- Provide game-level state access (difficulty, game type, scoring mode)
- Manage music playback via Lua
- Maintain compatibility layer for legacy Lua scripts

## External Dependencies
- **Core game interfaces:** `ActionQueues.h`, `player.h`, `monsters.h`, `projectiles.h`, `network.h`, `Music.h`, `screen.h`, `SoundManager.h`
- **Rendering/UI:** `game_window.h`, `Crosshairs.h`, `fades.h`, `ViewControl.h`
- **Lua C API:** `lua.h` (implicit via Lua includes)
- **Game data/definitions:** `map.h`, `item_definitions.h`, `projectile_definitions.h`
- **Game state:** `dynamic_world`, `static_world`, `current_player_index`, `local_player_index`, `GetGameQueue()`
- **Notable external functions:** `get_player_data()`, `GetGameQueue()`, `Crosshairs_IsActive()`, `SetTunnelVision()`, `SoundManager` methods, `Music::instance()`

# Source_Files/Lua/lua_player.h
## File Purpose
Declares the Lua scripting interface for player objects in Aleph One. Provides template-based bindings allowing Lua scripts to access individual players and iterate over the players collection via `Lua_Player` and `Lua_Players` classes.

## Core Responsibilities
- Define `Lua_Player` class wrapping player entity access for Lua
- Define `Lua_Players` container for iteration over all player objects
- Export the player registration function to set up Lua bindings
- Guard Lua-specific code behind compile-time feature flag

## External Dependencies
- **Lua 5.1**: `lua.h`, `lauxlib.h`, `lualib.h` (core C API)
- **Internal**: `lua_templates.h` (provides `L_Class<>` and `L_Container<>` template infrastructure)
- **Configuration**: `config.h` (compile-time `HAVE_LUA` guard)


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

## External Dependencies
- **Lua C API:** `lua.h`, `lauxlib.h`, `lualib.h`
- **Boost:** `boost/bind.hpp` (function binding)
- **Engine data structures:** `projectile_data`, `object_data`, `player_data`, `projectile_definition` (from `projectiles.h`, `map.h`, `player.h`, etc.)
- **Engine factories/accessors:** `new_projectile()`, `get_projectile_data()`, `get_object_data()`, `get_player_data()`, `get_projectile_definition()`, `play_object_sound()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`
- **Other Lua bindings:** `Lua_Monster`, `Lua_Player`, `Lua_Polygon`, `Lua_Sound`, `Lua_DamageType`, `Lua_ProjectileType` (from included headers)
- **Lua template system:** `L_Class`, `L_Enum`, `L_Container`, `L_EnumContainer`, `L_TableFunction` (from `lua_templates.h`)
- **Engine limits:** `get_dynamic_limit(_dynamic_limit_projectiles)` from `dynamic_limits.h`

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

## External Dependencies
- **Lua 5.1 API:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö core Lua C interface and auxiliary library functions  
- **Template implementations:** `lua_templates.h` ΓÇö defines `L_Class`, `L_Container`, `L_Enum`, `L_EnumContainer` template classes for binding C++ objects to Lua  
- **Core headers:** `cseries.h` (cross-platform utilities), `config.h` (build configuration)  
- **Defined elsewhere:** actual string values `Lua_Projectile_Name`, `Lua_Projectiles_Name`, `Lua_ProjectileType_Name`, `Lua_ProjectileTypes_Name` (implementation file); registration logic in corresponding `.cpp` file

# Source_Files/Lua/lua_script.cpp
## File Purpose
Controls the loading, execution, and event dispatching of embedded Lua scripts in the Aleph One game engine. Manages multiple script instances (embedded, network, solo), dispatches game events to Lua callbacks, and maintains script state across level transitions and saves.

## Core Responsibilities
- Load Lua scripts from buffers and manage Lua runtime lifecycle
- Dispatch game events to Lua trigger callbacks (player killed, item created, platform switches, damage, etc.)
- Register C++ functions exposed to Lua (player control, interface toggling, script termination)
- Maintain global script state (compass override, weapon wielding permissions, scoring mode)
- Serialize/deserialize script state for save games and level transitions
- Provide compatibility layer mapping new Lua-style trigger names to old function signatures
- Support conditional I/O access for solo vs. multiplayer script modes

## External Dependencies
- **Lua C API:** lua.h, lauxlib.h, lualib.h (Lua 5.1.2)
- **Game systems:** screen, player, render, weapons, monsters, world, network, physics
- **Lua binding modules:** lua_map, lua_monsters, lua_objects, lua_player, lua_projectiles
- **Boost:** shared_ptr (automatic cleanup), iostreams (binary serialization)
- **External symbols:** `ShootForTargetPoint()`, `get_physics_constants_for_model()`, `draw_panels()`, `screen_printf()`, `luaopen_*` (Lua standard libraries)

# Source_Files/Lua/lua_script.h
## File Purpose
Header file declaring the Lua scripting system interface for Aleph One game engine. Provides lifecycle management for Lua scripts, event callbacks for game interactions, camera/cutscene path definitions, and queries for Lua-controlled game state.

## Core Responsibilities
- Lua script lifecycle (load, execute, unload, error handling)
- Event dispatch to Lua for game events (switches, terminals, damage, kills, item pickups)
- Invalidation signals when entities are destroyed (monsters, projectiles, objects)
- Camera/cutscene path system with timed keyframes for position and rotation
- Texture palette management for Lua-controlled textures
- Game mode queries (scoring mode, end conditions, weapon availability)
- State persistence (serialize/deserialize Lua state for save/load)
- Action queue access for scripted player input
- Script muting and collection memory marking

## External Dependencies
- `config.h` ΓÇö feature flags (HAVE_LUA guard)
- `cseries.h` ΓÇö core types, macros, utilities
- `world.h` ΓÇö world geometry (`world_point3d`, `angle`)
- `ActionQueues.h` ΓÇö player input queue system (`ActionQueues` class)
- `shape_descriptors.h` ΓÇö texture/shape handles (`shape_descriptor` typedef)
- Standard library: `<cstdint>` (`int32`, `uint8`, `short`), `<string>` (ExecuteLuaString parameter)

# Source_Files/Lua/lua_serialize.cpp
## File Purpose
Serializes and deserializes Lua objects (tables, numbers, strings, userdata, etc.) to/from binary streams. Supports version-controlled binary format with reference tracking to handle shared and recursive structures. Entry points are `lua_save()` and `lua_restore()` for use by the Lua runtime.

## Core Responsibilities
- Recursively serialize Lua values (nil, number, boolean, string, table, userdata) to binary format
- Track object references to avoid duplicating shared values and handle cyclic references
- Validate table keys to ensure only serializable types are used
- Restore Lua values from binary stream in matching format
- Handle userdata by storing metatable name and index; reconstruct via `__new` metamethod
- Manage version checking during deserialization; reject newer formats
- Log errors on I/O failures during serialization/deserialization

## External Dependencies
- **Lua C API**: `lua.h`, `lauxlib.h`, `lualib.h` (lua_State, lua_pushvalue, lua_rawget, lua_type, lua_tonumber, lua_tostring, etc.)
- **Serialization**: `BStream.h` ΓÇö `BOStreamBE` (output stream), `BIStreamBE` (input stream), `basic_bstream::failure` exception
- **Logging**: `Logging.h` ΓÇö `logWarning()`, `logWarning1()` for error reporting
- **Standard library**: `<vector>` for buffering; `<streambuf>` for stream abstraction
- **Config**: `config.h` ΓÇö conditional compilation guard `HAVE_LUA`

# Source_Files/Lua/lua_serialize.h
## File Purpose
Declares a serialization interface for Lua objects. Provides functions to save Lua values from the interpreter stack to a C++ stream buffer and restore them back, enabling persistence of game state or configuration data. Bridges Lua's C API with C++ stream utilities.

## Core Responsibilities
- Declare `lua_save()` to serialize Lua stack objects to streams
- Declare `lua_restore()` to deserialize and reconstruct Lua objects
- Provide conditional compilation for Lua support (HAVE_LUA guard)
- Expose a C++ standard library interface (std::streambuf) for Lua serialization

## External Dependencies
- `config.h` ΓÇö Feature detection (HAVE_LUA)
- `cseries.h` ΓÇö Engine series library
- `<streambuf>` ΓÇö C++ standard stream buffering
- Lua C API (wrapped in `extern "C"`): `lua.h`, `lauxlib.h`, `lualib.h`
- Function implementations defined elsewhere (not in this header)

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

## External Dependencies
- **Lua C API** (`lua.h`, `lauxlib.h`, `lualib.h`): Stack manipulation, type checking, table/registry access, userdata creation
- **Boost** (`boost/function.hpp`): Function object wrapper for validation and length callbacks (allows both functor and function pointers)
- **STL** (`<sstream>`, `<map>`, `<string>`): String formatting, mnemonic lookups
- **Engine** (`lua_script.h`, `lua_mnemonics.h`): Game-specific callbacks, enum definitions; `L_Persistent_Table_Key()` function (defined elsewhere)
- **C series** (`cseries.h`): Platform abstraction, type definitions

**Note**: Actual getter/setter implementations, validation functors, and container types (T) are provided by code that instantiates these templates.

# Source_Files/Lua/luaconf.h
## File Purpose
Configuration header for Lua interpreter/VM. Defines platform-specific preprocessor macros that customize Lua's behavior, type system, stack limits, and module loading paths without modifying core source files.

## Core Responsibilities
- Platform detection (Windows, POSIX/Linux, macOS) and feature flag setup
- Module and C library search paths (`LUA_PATH`, `LUA_CPATH`)
- Type configuration (integer width, number format, memory types)
- API symbol visibility and calling convention (`LUA_API`, `LUALIB_API`)
- Stack/recursion limits and array bounds (`LUAI_MAXCALLS`, `LUAI_MAXVARS`, etc.)
- Garbage collector tuning (`LUAI_GCPAUSE`, `LUAI_GCMUL`)
- Numeric operations (double-precision arithmetic macros)
- Dynamic library loading method selection
- Exception handling mechanism (C++/longjmp/setjmp)
- Backward compatibility flags for deprecated Lua 5.0 features

## External Dependencies
- Standard C: `<limits.h>`, `<stddef.h>`, `<math.h>`, `<assert.h>`, `<unistd.h>`, `<io.h>`, `<stdio.h>`
- Optional readline: `<readline/readline.h>`, `<readline/history.h>` (if `LUA_USE_READLINE`)
- Generated config: `config.h` (from bundled context, defines `HAVE_OPENGL`, `HAVE_SDL_IMAGE`, `HAVE_SDL_NET`, etc.)
- Platform-specific symbols (not included): Windows `_isatty()`, `_fileno()`, `_popen()`; Unix `isatty()`, `popen()`, `mkstemp()`, `dlopen()`

# Source_Files/Lua/lualib.h
## File Purpose
Header declaring Lua standard library initialization functions for the game engine's scripting system. Provides function declarations to open/register Lua's built-in modules (base, table, I/O, OS, string, math, debug, package) into a Lua state. Wrapped in `HAVE_LUA` guard, indicating scripting is an optional feature.

## Core Responsibilities
- Declare library initialization functions (`luaopen_*`) for each Lua standard library module
- Define symbolic names (macros) for library identifiers used in registration
- Declare `luaL_openlibs()` as a convenience function to load all libraries at once
- Define file-handle type key for Lua's I/O library
- Provide a default `lua_assert` macro if not already defined by the host

## External Dependencies
- **Includes**: `config.h` (build-time feature flags), `lua.h` (core Lua C API, defines `lua_State`)
- **External symbols**: `lua_State` (opaque Lua execution context, defined in lua.h)
- **LUALIB_API**: Macro controlling visibility of library functions (likely defined in luaconf.h, included via lua.h)

# Source_Files/Lua/lundump.h
## File Purpose
Header file providing the interface for Lua binary chunk serialization and deserialization. Declares functions to load precompiled bytecode from binary streams, create binary file headers, and dump function prototypes to binary format for caching or distribution.

## Core Responsibilities
- Declare loader interface for deserializing Lua bytecode from binary (undump)
- Declare dumper interface for serializing Lua function prototypes to binary (dump)
- Declare binary header generation interface for chunk files
- Declare debug/inspection printer for bytecode introspection (conditional)
- Define constants for Lua binary format versioning and structure sizes

## External Dependencies
- `config.h` ΓÇö `HAVE_LUA` preprocessor guard
- `lobject.h` ΓÇö Proto struct definition, GCObject types
- `lzio.h` ΓÇö ZIO and Mbuffer structures
- `lua.h` ΓÇö lua_State opaque type, lua_Writer callback typedef, LUAI_FUNC macro (defined elsewhere)

# Source_Files/Lua/lvm.h
## File Purpose
Header for the Lua virtual machine. Declares the core VM execution loop and runtime value operations (type conversion, comparison, table access, string concatenation). This is the primary interface for executing Lua bytecode and manipulating Lua values at runtime.

## Core Responsibilities
- Declare the main VM execution entry point (`luaV_execute`)
- Provide type checking and conversion macros for runtime type coercion
- Declare comparison and equality operators for Lua values
- Declare table access operations (get/set) invoked during code execution
- Declare string concatenation for the CONCAT bytecode operation
- Define helper macros for safe type conversions with side effects

## External Dependencies
- **Includes**: 
  - `config.h` (compile-time configuration)
  - `ldo.h` (stack/call structure)
  - `lobject.h` (Lua object types and TValue definition)
  - `ltm.h` (tag methods/metamethods)
- **External symbols** (defined elsewhere):
  - `lua_State` (Lua execution state, opaque)
  - `TValue`, `StkId` (object types from lobject.h)
  - Tag method functions from ltm.h (for metamethod lookup)

# Source_Files/Lua/lzio.h
## File Purpose

Header for Lua's buffered stream (ZIO) I/O system. Provides macros and function declarations for reading data from custom sources (files, memory, network, etc.) via a reader callback pattern, with support for dynamic buffering.

## Core Responsibilities

- Define the `Zio` buffered stream struct for reading from custom sources
- Define the `Mbuffer` dynamic memory buffer struct
- Provide macros for buffer access, resizing, and character extraction
- Declare initialization and reading functions
- Support the reader callback pattern for flexible input sources

## External Dependencies

- `lua.h`: Defines `lua_State`, `lua_Reader` callback typedef
- `lmem.h`: Memory allocation/reallocation macros (`luaM_reallocvector`, `luaM_toobig`)
- `config.h`: Conditional compilation (e.g., `HAVE_LUA`)


