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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Action_Flags` | L_Class template | Wraps action button state flags (trigger, weapon cycle, map toggle, etc.) |
| `Lua_Camera` | L_Class template | Camera object with path animation capability |
| `Lua_Player` | L_Class template | Player data wrapper (position, energy, items, weapons, etc.) |
| `Lua_Players` | L_Container template | Collection container for all active players |
| `Lua_Game` | L_Class template | Global game state (difficulty, type, time, scoring) |
| `Lua_Music` | L_Class template | Music playback controller |
| `PlayerSubtable<name>` | Template class | Generic sub-object bound to a specific player (overlays, compass, items, velocity) |
| `lua_camera` (external) | struct | Stores camera path data and active state (defined elsewhere) |
| `player_data` (external) | struct | Core player state (location, items, energy, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `lua_cameras` | `vector<lua_camera>` | extern global | All camera objects managed by the engine |
| `number_of_cameras` | `int` | extern global | Count of active cameras |
| `game_end_condition` | `int` | extern global | Game termination state flag |
| `game_scoring_mode` | `int` | extern global | Current scoring rules mode |
| `lua_random_generator` | `GM_Random` | extern global | Deterministic RNG for Lua |
| `use_lua_compass[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | `bool[]` | extern global | Per-player compass override flag |
| `lua_compass_beacons[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | `world_point2d[]` | extern global | Per-player beacon positions |
| `lua_compass_states[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | `short[]` | extern global | Per-player compass direction bits |
| `local_player_index` | `int` | extern global | Current human player index |
| `current_player_index` | `int` | extern global | Active player for network updates |

## Key Functions / Methods

### Lua_Player_register
- **Signature:** `int Lua_Player_register(lua_State *L)`
- **Purpose:** Master registration function that binds all player-related Lua classes to the VM. Called during Lua environment initialization.
- **Inputs:** Lua state pointer
- **Outputs/Return:** Returns 0 on success
- **Side effects:** Registers 15+ Lua classes with getter/setter tables; sets global "Game" and "Music" objects; loads compatibility layer
- **Calls:** `L_Class::Register()`, `L_Container::Register()`, `L_Enum::Register()`, `Lua_Player_load_compatibility()`
- **Notes:** Must be called exactly once before any Lua script accesses player APIs. Global objects created at end persist for game session.

### Lua_Action_Flags_Get_t / Lua_Action_Flags_Set_t (templates)
- **Signature:** `static int Lua_Action_Flags_Get_t<flag>(lua_State *L)` / `static int Lua_Action_Flags_Set_t<flag>(lua_State *L)`
- **Purpose:** Template functions for reading/writing individual action flag bits (trigger, weapon cycle, map toggle). Used during idle() callback to simulate player input.
- **Inputs:** Lua state; L(1) = player index; L(2) = boolean value (set only)
- **Outputs/Return:** Boolean flag state (get) or 0 (set)
- **Side effects:** Modifies queued action flags via `GetGameQueue()->modifyActionFlags()`
- **Calls:** `GetGameQueue()->countActionFlags()`, `GetGameQueue()->peekActionFlags()`, `GetGameQueue()->modifyActionFlags()`
- **Notes:** Only callable during idle() callback; returns error if action queue empty. Uses flag template parameter to act on specific bit masks.

### Lua_Camera_Activate
- **Signature:** `int Lua_Camera_Activate(lua_State *L)`
- **Purpose:** Activate a camera view for a player, initializing playback of stored path animation.
- **Inputs:** Lua state; L(1) = camera object; L(2) = player index (number or Player object)
- **Outputs/Return:** 0
- **Side effects:** Resets camera elapsed time, path indices, and timestamps; only applies if player is local player
- **Calls:** None (direct struct access)
- **Notes:** Only affects local_player_index; networked players do not see camera animations. Path playback begins at point/angle index 0.

### Lua_Player_Set_Energy
- **Signature:** `static int Lua_Player_Set_Energy(lua_State *L)`
- **Purpose:** Set player shield/energy level, clamped to maximum suit capacity.
- **Inputs:** Lua state; L(1) = player index; L(2) = energy value (number)
- **Outputs/Return:** 0
- **Side effects:** Updates suit energy; calls `mark_shield_display_as_dirty()` to queue UI redraw
- **Calls:** `get_player_data()`, `mark_shield_display_as_dirty()`
- **Notes:** Clamps input to 3├ù PLAYER_MAXIMUM_SUIT_ENERGY. Alias "juice" and "life" map to this function.

### Lua_Game_Get_Time_Remaining
- **Signature:** `static int Lua_Game_Get_Time_Remaining(lua_State* L)`
- **Purpose:** Return remaining match time in ticks, or nil if timer disabled/very high.
- **Inputs:** Lua state
- **Outputs/Return:** Number (ticks) or nil
- **Side effects:** None
- **Calls:** None
- **Notes:** Returns nil if time > 999*30 (disabled indicator). Used by scripts for time-based logic.

## Control Flow Notes
This file is Lua API glueΓÇöit has no frame/update/render logic of its own. Functions are invoked exclusively by the Lua VM in response to script calls. Registration happens during engine initialization (before Lua scripts run). Action flags are processed during `idle()` callback in the main game loop. Camera and overlay updates happen in the render phase (external to this file).

## External Dependencies
- **Core game interfaces:** `ActionQueues.h`, `player.h`, `monsters.h`, `projectiles.h`, `network.h`, `Music.h`, `screen.h`, `SoundManager.h`
- **Rendering/UI:** `game_window.h`, `Crosshairs.h`, `fades.h`, `ViewControl.h`
- **Lua C API:** `lua.h` (implicit via Lua includes)
- **Game data/definitions:** `map.h`, `item_definitions.h`, `projectile_definitions.h`
- **Game state:** `dynamic_world`, `static_world`, `current_player_index`, `local_player_index`, `GetGameQueue()`
- **Notable external functions:** `get_player_data()`, `GetGameQueue()`, `Crosshairs_IsActive()`, `SetTunnelVision()`, `SoundManager` methods, `Music::instance()`
