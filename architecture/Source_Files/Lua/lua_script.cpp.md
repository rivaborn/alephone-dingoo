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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `LuaState` | class | Base class managing a Lua runtime with trigger dispatch, function registration, and state persistence |
| `SoloScriptState` | class | Specialized LuaState allowing I/O library access for solo debug scripts |
| `EmbeddedLuaState` | typedef | Type alias for LuaState used in embedded level scripts |
| `NetscriptState` | typedef | Type alias for LuaState used in network game scripts |
| `luaL_Reg` | struct (from lauxlib.h) | Name-function pairs for registering C functions with Lua |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `use_lua_compass` | bool[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS] | global | Per-player flag to override compass with Lua beacon |
| `can_wield_weapons` | bool[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS] | global | Per-player weapon wielding enable flag |
| `lua_compass_beacons` | world_point2d[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS] | global | Beacon positions for Lua-controlled compass |
| `game_scoring_mode` | int | global | Current game scoring mode (e.g., _game_of_most_points) |
| `game_end_condition` | int | global | Game end condition set by script (e.g., _game_normal_end_condition) |
| `sLuaActionQueues` | static ActionQueues* | static | Queues for Lua-controlled player actions |
| `mute_lua` | static bool | static | Mutes Lua script debug messages |
| `PassedLuaState` | std::map<int, std::string> | static | Script states passed to next level |
| `SavedLuaState` | std::map<int, std::string> | static | Script states saved for restoration |
| `states` | static std::map (type ΓåÆ LuaState) | static | Map of script types to active Lua runtime instances |

## Key Functions / Methods

### LoadLuaScript (global)
- **Signature:** `bool LoadLuaScript(const char *buffer, size_t len, ScriptType script_type)`
- **Purpose:** Parse and load Lua script source code into the appropriate script state instance.
- **Inputs:** buffer (script source), len (byte count), script_type (embedded/netscript/solo)
- **Outputs/Return:** true if load succeeded (status==0)
- **Side effects:** Creates LuaState instance if needed, calls `Initialize()` on first use
- **Calls:** `LuaState::Load()`, `LuaState::Initialize()`
- **Notes:** Initializes compatibility layer and registers all C functions

### RunLuaScript (global)
- **Signature:** `bool RunLuaScript()`
- **Purpose:** Execute all loaded scripts and initialize the Lua environment for gameplay.
- **Inputs:** None (reads global script map)
- **Outputs/Return:** true if any script runs successfully
- **Side effects:** Calls `Init(false)` trigger on all scripts; resets compass/weapon/scoring state
- **Calls:** `LuaState::Run()`, `PreservePreLuaSettings()`, `InitializeLuaVariables()`

### L_Call_* family (global, ~20 functions)
- **Signature:** `void L_Call_<TriggerName>(<event args>)`
- **Purpose:** Dispatch game events to all active Lua scripts.
- **Inputs:** Event-specific (e.g., player_index, projectile_index, damage_type)
- **Outputs/Return:** None (void)
- **Side effects:** Calls corresponding `LuaState::` trigger method on each active script
- **Calls:** Iterates `states` map, calls trigger methods
- **Examples:** `L_Call_Player_Killed()`, `L_Call_Got_Item()`, `L_Call_Idle()`, `L_Call_Tag_Switch()`

### LuaState::GetTrigger
- **Signature:** `bool LuaState::GetTrigger(const char *trigger)`
- **Purpose:** Look up and push a named trigger function from the Triggers table.
- **Inputs:** trigger name string (e.g., "init", "player_killed")
- **Outputs/Return:** true if function found and pushed on stack; false otherwise
- **Side effects:** Pops stack if function not found; leaves function on stack if found
- **Calls:** Lua C API: `lua_pushstring()`, `lua_gettable()`, `lua_isfunction()`, `lua_pop()`, `lua_remove()`
- **Notes:** Checks `running_` flag; returns false if not running

### LuaState::CallTrigger
- **Signature:** `void LuaState::CallTrigger(int numArgs=0)`
- **Purpose:** Execute a trigger function with error handling.
- **Inputs:** numArgs (number of arguments already on stack)
- **Outputs/Return:** None (void)
- **Side effects:** Calls Lua function; on error, calls `L_Error()` with error message
- **Calls:** `lua_pcall()`, `L_Error()`

### CloseLuaScript (global)
- **Signature:** `void CloseLuaScript()`
- **Purpose:** Clean up all Lua scripts and save state for level transitions.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Saves all script states to `PassedLuaState`, clears `states` map, resets camera/muting/scoring state
- **Calls:** `LuaState::SavePassed()`, `RestorePreLuaSettings()`, `LuaTexturePaletteClear()`

### L_Player_Control (C function registered with Lua)
- **Signature:** `static int L_Player_Control(lua_State *L)`
- **Purpose:** Allow Lua scripts to manipulate player movement, looking, and action triggers.
- **Inputs:** Lua stack: player_index, move_type, value, optional args (x, y, z for movement)
- **Outputs/Return:** 0
- **Side effects:** Creates `sLuaActionQueues` if needed; enqueues action flags for player
- **Calls:** `GetLuaActionQueues()->enqueueActionFlags()`, `get_physics_constants_for_model()`
- **Notes:** Supports simple action queue (move_forward, turn_left, etc.) and complex TIENNOU_PLAYER_CONTROL mode with pathfinding

### save_lua_states / pack_lua_states / unpack_lua_states (global)
- **Purpose:** Serialize/deserialize all script persistent state for multiplayer save/load.
- **Inputs/Outputs:** Binary format with script type ID and length prefix per script
- **Calls:** Stream I/O with boost::iostreams; `LuaState::SaveAll()`

## Control Flow Notes
- **Init:** `LoadLuaScript()` ΓåÆ `RunLuaScript()` ΓåÆ calls `init` trigger
- **Per-frame:** `L_Call_Idle()` ΓåÆ calls `idle` trigger
- **Post-frame:** `L_Call_PostIdle()` ΓåÆ calls `postidle` trigger
- **Events:** Various `L_Call_*` dispatch to corresponding trigger (e.g., player death, item pickup)
- **Shutdown:** `CloseLuaScript()` saves state, clears runtime

## External Dependencies
- **Lua C API:** lua.h, lauxlib.h, lualib.h (Lua 5.1.2)
- **Game systems:** screen, player, render, weapons, monsters, world, network, physics
- **Lua binding modules:** lua_map, lua_monsters, lua_objects, lua_player, lua_projectiles
- **Boost:** shared_ptr (automatic cleanup), iostreams (binary serialization)
- **External symbols:** `ShootForTargetPoint()`, `get_physics_constants_for_model()`, `draw_panels()`, `screen_printf()`, `luaopen_*` (Lua standard libraries)
