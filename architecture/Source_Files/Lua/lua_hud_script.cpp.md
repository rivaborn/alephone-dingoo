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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `LuaHUDState` | class | Encapsulates Lua VM state, script loading, trigger dispatch, and resource tracking. Manages lifecycle of lua_State via shared_ptr. |
| `lualibs[]` | static array | Table of standard Lua library open functions (base, table, io, os, string, math, debug). |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `hud_state` | `LuaHUDState*` | global | Singleton pointer to the active HUD Lua state; NULL if no script loaded. |
| `AngleConvert` | `float` | extern (global) | Game engine constant for angle conversion; accessed by HUD scripts. |
| `MotionSensorActive` | `bool` | extern (global) | Game state flag; accessible to HUD scripts. |
| `world_view` | `view_data*` | extern (global) | Current camera/view data; used for HUD rendering context. |
| `static_world` | `static_data*` | extern (global) | Game world static data; referenced during HUD operations. |

## Key Functions / Methods

### LuaHUDState::Load
- **Signature:** `bool Load(const char *buffer, size_t len)`
- **Purpose:** Parse and load Lua bytecode/source from a memory buffer without executing it.
- **Inputs:** `buffer` (pointer to script data), `len` (size in bytes)
- **Outputs/Return:** `true` if load succeeded (status == 0), `false` if parse/memory error
- **Side effects:** Increments `num_scripts_` counter on success; pushes function onto Lua stack.
- **Calls:** `luaL_loadbuffer()`, `logWarning()`
- **Notes:** Logs detailed error messages (syntax, memory, file, runtime). Does not execute yet.

### LuaHUDState::Run
- **Signature:** `bool Run()`
- **Purpose:** Execute all loaded scripts in the Lua state and set running flag if successful.
- **Inputs:** None (uses state of `num_scripts_` counter)
- **Outputs/Return:** `true` if all scripts executed without error, `false` if pcall failed
- **Side effects:** Sets `running_ = true` on success; manipulates Lua stack (lua_insert, lua_pcall).
- **Calls:** `lua_insert()`, `lua_pcall()`
- **Notes:** Reverses function stack order before execution. Returns `false` if `!Loaded()`.

### LuaHUDState::GetTrigger
- **Signature:** `bool GetTrigger(const char *trigger)`
- **Purpose:** Look up a named trigger function in the Lua global `Triggers` table.
- **Inputs:** `trigger` (name of callback, e.g., "init", "draw")
- **Outputs/Return:** `true` if function found and pushed onto stack; `false` if not found or not a function
- **Side effects:** Leaves function on top of Lua stack (or pops stack if failed).
- **Calls:** `lua_pushstring()`, `lua_gettable()`, `lua_istable()`, `lua_isfunction()`, `lua_pop()`, `lua_remove()`
- **Notes:** Returns `false` immediately if `!running_`. Returns false if `Triggers` table not found.

### LuaHUDState::CallTrigger
- **Signature:** `void CallTrigger(int numArgs = 0)`
- **Purpose:** Execute a Lua function that has been pushed onto the stack, with error reporting.
- **Inputs:** `numArgs` (number of arguments on stack; default 0)
- **Outputs/Return:** None (void)
- **Side effects:** Executes Lua code; calls `L_Error()` if runtime error occurs.
- **Calls:** `lua_pcall()`, `lua_tostring()`, `L_Error()`
- **Notes:** Always expects 0 return values (nresults=0). Suppresses stack trace on LUA_ERRRUN.

### LuaHUDState::Init / Draw / Resize / Cleanup
- **Signature:** `void Init()`, `void Draw()`, `void Resize()`, `void Cleanup()`
- **Purpose:** Invoke the corresponding Lua trigger callback (init, draw, resize, cleanup).
- **Inputs:** None (closure over `this`)
- **Outputs/Return:** None (void)
- **Side effects:** Calls Lua functions; modifies `inited_` flag; may log errors.
- **Calls:** `GetTrigger()`, `CallTrigger()`
- **Notes:** `Draw()`, `Resize()`, `Cleanup()` return early if `!inited_`. `Init()` sets `inited_ = true`. `Cleanup()` sets `inited_ = false`.

### LuaHUDState::RegisterFunctions
- **Signature:** `void RegisterFunctions()`
- **Purpose:** Register C++ functions exported to Lua scripts for HUD object manipulation.
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Modifies Lua global table to add exported functions.
- **Calls:** `Lua_HUDObjects_register()`
- **Notes:** Called from `Initialize()`. Actual exports defined in `lua_hud_objects.h`.

### LuaHUDState::MarkCollections
- **Signature:** `void MarkCollections(std::set<short>& collections)`
- **Purpose:** Read declared collection indices from Lua global `CollectionsUsed` and mark them for loading.
- **Inputs:** `collections` (output set to populate with collection indices)
- **Outputs/Return:** None (void); populates `collections` set as side effect
- **Side effects:** Calls `mark_collection_for_loading()` for each valid index. Returns early if `!running_`.
- **Calls:** `lua_pushstring()`, `lua_gettable()`, `lua_istable()`, `lua_isnumber()`, `lua_tonumber()`, `lua_pushnumber()`, `lua_pop()`, `mark_collection_for_loading()`
- **Notes:** Handles both table (array) and scalar number formats. Validates collection_index against `NUMBER_OF_COLLECTIONS`.

### LoadLuaHUDScript
- **Signature:** `bool LoadLuaHUDScript(const char *buffer, size_t len)`
- **Purpose:** Public API to load a Lua HUD script. Creates singleton if needed.
- **Inputs:** `buffer` (script data), `len` (size)
- **Outputs/Return:** `true` if load successful
- **Side effects:** Creates global `hud_state` if null; calls `Initialize()` on new state.
- **Calls:** `LuaHUDState::Load()`
- **Notes:** Lazy initialization of singleton.

### RunLuaHUDScript
- **Signature:** `bool RunLuaHUDScript()`
- **Purpose:** Public API to execute loaded scripts.
- **Inputs:** None
- **Outputs/Return:** `true` if execution succeeded
- **Side effects:** None (delegates to `hud_state->Run()`)
- **Calls:** `LuaHUDState::Run()`
- **Notes:** Returns `false` if `hud_state` is null.

### LoadHUDLua
- **Signature:** `void LoadHUDLua()`
- **Purpose:** Load HUD Lua script from file specified in game preferences.
- **Inputs:** None (reads from `environment_preferences->hud_lua_file`)
- **Outputs/Return:** None (void)
- **Side effects:** Opens file, reads into buffer, calls `LoadLuaHUDScript()`. Preserves prior game error state.
- **Calls:** `FileSpecifier()`, `OpenedFile::Open()`, `OpenedFile::GetLength()`, `OpenedFile::Read()`, `LoadLuaHUDScript()`, `get_game_error()`, `set_game_error()`
- **Notes:** Only runs if `environment_preferences->use_hud_lua` is true. Guards Lua errors from interfering with error checking infrastructure.

### CloseLuaHUDScript
- **Signature:** `void CloseLuaHUDScript()`
- **Purpose:** Destroy the Lua HUD state singleton.
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Deletes global `hud_state` and sets to NULL.
- **Calls:** (destructor of LuaHUDState is implicit)
- **Notes:** Called during game shutdown.

### MarkLuaHUDCollections
- **Signature:** `void MarkLuaHUDCollections(bool loading)`
- **Purpose:** Public API to enable/disable collection loading tracking based on Lua script declarations.
- **Inputs:** `loading` (true to collect indices; false to unload them)
- **Outputs/Return:** None (void)
- **Side effects:** On `loading=true`, clears static set and populates via `hud_state->MarkCollections()`. On `loading=false`, unloads all tracked collections.
- **Calls:** `LuaHUDState::MarkCollections()`, `mark_collection_for_unloading()`
- **Notes:** Uses static set to remember which collections were marked across loading/unloading phases.

### Public C-linkage stubs (when HAVE_LUA undefined)
- `void L_Call_HUDInit()`, `L_Call_HUDCleanup()`, `L_Call_HUDDraw()`, `L_Call_HUDResize()`
- Empty no-op implementations when Lua is not available.

## Control Flow Notes
- **Conditional Compilation:** The file provides two implementations: stubs when `HAVE_LUA` is undefined, and full Lua integration when `HAVE_LUA` is defined.
- **Singleton Pattern:** Global `hud_state` is lazily created on first script load and destroyed on shutdown.
- **Lifecycle:** 
  - Script load via `LoadHUDLua()` ΓåÆ `LoadLuaHUDScript()` ΓåÆ `LuaHUDState::Load()`
  - Script execution via `RunLuaHUDScript()` ΓåÆ `LuaHUDState::Run()`
  - HUD callbacks invoked by engine via `L_Call_HUDInit/Draw/Resize/Cleanup()` ΓåÆ `hud_state->Init/Draw/Resize/Cleanup()` ΓåÆ `GetTrigger()` + `CallTrigger()`
  - Collection tracking via `MarkLuaHUDCollections()` before level load, then unmark after unload
- **Error Handling:** Lua runtime errors are caught by `lua_pcall()` and logged. File load errors are isolated from game error state.

## External Dependencies
- **Lua 5.1.2 libraries:** `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö VM, auxiliary functions, standard library bindings
- **Boost:** `shared_ptr` (RAII wrapper for lua_State), `iostreams` (unused in this file but included)
- **Game engine:** `preferences.h` (environment_preferences), `interface.h` (FileSpecifier, mark_collection_for_loading/unloading, game error state), `mouse.h`, `Logging.h` (logWarning)
- **HUD bindings:** `lua_hud_objects.h` ΓåÆ `Lua_HUDObjects_register()` (defined elsewhere)
- **Externs defined elsewhere:** `AngleConvert`, `MotionSensorActive`, `world_view`, `static_world`, `L_Error()`, `L_Persistent_Table_Key()`
