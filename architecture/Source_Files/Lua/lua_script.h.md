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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ScriptType` | enum | Distinguishes embedded (in-map), netscript (network multiplayer), and solo (campaign) Lua scripts |
| `timed_point` | struct | 3D world position with polygon and millisecond timing delta for camera path keyframes |
| `timed_angle` | struct | Yaw/pitch rotation pair with millisecond timing delta for camera orientation keyframes |
| `lua_path` | struct | Complete camera path with vectors of position and rotation keyframes, current index, and last-frame timestamps |
| `lua_camera` | struct | Active camera instance with associated path, elapsed time, and active player index |

Game scoring/end-condition enums: `_game_of_most_points`, `_game_no_end_condition`, etc.

## Global / File-Static State
None (header file only; state managed in implementation).

## Key Functions / Methods

### L_Call_Init
- **Signature:** `void L_Call_Init(bool fRestoringSaved)`
- **Purpose:** Initialize Lua VM and load initial scripts when level/game starts
- **Inputs:** `fRestoringSaved` ΓÇö whether reloading from saved game
- **Side effects:** Initializes Lua state, executes init scripts

### L_Call_Cleanup
- **Signature:** `void L_Call_Cleanup()`
- **Purpose:** Shut down Lua VM and unload scripts
- **Side effects:** Deallocates Lua state and resources

### LoadLuaScript
- **Signature:** `bool LoadLuaScript(const char *buffer, size_t len, ScriptType type)`
- **Purpose:** Load a Lua script from memory buffer
- **Inputs:** Script buffer, length, and type (embedded/netscript/solo)
- **Outputs:** True if loaded successfully

### RunLuaScript
- **Signature:** `bool RunLuaScript()`
- **Purpose:** Execute the loaded Lua script
- **Outputs:** True if executed without error

### L_Call_Idle / L_Call_PostIdle
- **Purpose:** Called each frame; allows Lua scripts to update state and check conditions

### Event Callbacks (L_Call_*)
Game events dispatch to Lua via these callbacks (e.g., `L_Call_Tag_Switch`, `L_Call_Player_Killed`, `L_Call_Projectile_Detonated`). Each receives entity indices and relevant contextual parameters.

### Invalidation Functions
`L_Invalidate_Monster`, `L_Invalidate_Projectile`, `L_Invalidate_Object` ΓÇö notify Lua that entities have been destroyed so Lua can clear references.

### State Persistence
- `save_lua_states()` ΓÇö serialize Lua state; returns buffer size
- `unpack_lua_states(uint8* data, size_t length)` ΓÇö restore from save buffer
- `pack_lua_states(uint8* data, size_t length)` ΓÇö write current state to buffer

### Texture Palette
`LuaTexturePaletteClear`, `LuaTexturePaletteSize`, `LuaTexturePaletteTexture`, `LuaTexturePaletteTextureType`, `LuaTexturePaletteSelected` ΓÇö manage texture collection for Lua-controlled rendering.

### Query Functions
- `GetLuaScoringMode()` / `GetLuaGameEndCondition()` ΓÇö retrieve Lua-controlled game settings
- `UseLuaCameras()` ΓÇö check if active camera should use Lua path system
- `LuaPlayerCanWieldWeapons(short player_index)` ΓÇö Lua control over weapon availability

## Control Flow Notes
Scripts follow game lifecycle:
1. **Init:** `L_Call_Init()` on level start
2. **Main loop:** `L_Call_Idle()` ΓåÆ game updates ΓåÆ `L_Call_PostIdle()` each frame
3. **Event dispatch:** Game events (switches, kills, etc.) call corresponding `L_Call_*` callbacks
4. **Cleanup:** `L_Call_Cleanup()` on level end; `CloseLuaScript()` on engine shutdown

Camera system: Scripts populate `lua_camera` with keyframed paths; `UseLuaCameras()` gates whether renderer uses them.

## External Dependencies
- `config.h` ΓÇö feature flags (HAVE_LUA guard)
- `cseries.h` ΓÇö core types, macros, utilities
- `world.h` ΓÇö world geometry (`world_point3d`, `angle`)
- `ActionQueues.h` ΓÇö player input queue system (`ActionQueues` class)
- `shape_descriptors.h` ΓÇö texture/shape handles (`shape_descriptor` typedef)
- Standard library: `<cstdint>` (`int32`, `uint8`, `short`), `<string>` (ExecuteLuaString parameter)
