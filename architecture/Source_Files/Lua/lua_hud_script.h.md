# Source_Files/Lua/lua_hud_script.h

## File Purpose
Header file declaring the Lua HUD scripting interface for Aleph One. Provides callback entry points for Lua scripts to hook into the game's HUD lifecycle (init, cleanup, draw, resize) and functions to load/unload Lua HUD scripts.

## Core Responsibilities
- Declare callback functions triggered by the HUD system (init, cleanup, draw, resize)
- Provide script loading/execution interface (`LoadLuaHUDScript`, `RunLuaHUDScript`)
- Manage Lua HUD script lifecycle (load, run, close)
- Support memory management for Lua collections during loading
- Bridge C/C++ game engine with Lua HUD scripts

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### L_Call_HUDInit
- Signature: `void L_Call_HUDInit()`
- Purpose: Trigger HUD initialization callback in Lua
- Inputs: None
- Outputs/Return: None
- Side effects: Calls Lua HUD init function if defined
- Calls: (not visible in this file)
- Notes: Part of lifecycle callback pattern

### L_Call_HUDCleanup
- Signature: `void L_Call_HUDCleanup()`
- Purpose: Trigger HUD cleanup callback in Lua
- Inputs: None
- Outputs/Return: None
- Side effects: Calls Lua HUD cleanup function if defined
- Calls: (not visible in this file)
- Notes: Part of lifecycle callback pattern

### L_Call_HUDDraw
- Signature: `void L_Call_HUDDraw()`
- Purpose: Trigger HUD rendering callback in Lua
- Inputs: None
- Outputs/Return: None
- Side effects: Calls Lua HUD draw function (likely per-frame)
- Calls: (not visible in this file)
- Notes: Called during rendering phase

### L_Call_HUDResize
- Signature: `void L_Call_HUDResize()`
- Purpose: Trigger HUD resize callback in Lua
- Inputs: None
- Outputs/Return: None
- Side effects: Calls Lua HUD resize function on resolution change
- Calls: (not visible in this file)
- Notes: Called when display/window resizes

### LoadLuaHUDScript
- Signature: `bool LoadLuaHUDScript(const char *buffer, size_t len)`
- Purpose: Parse and load a Lua HUD script from memory buffer
- Inputs: `buffer` (script text), `len` (buffer length)
- Outputs/Return: `bool` (success/failure)
- Side effects: Initializes Lua HUD state, may allocate memory
- Calls: (not visible in this file)
- Notes: Not inferable whether this compiles or just loads raw script

### RunLuaHUDScript
- Signature: `bool RunLuaHUDScript()`
- Purpose: Execute the loaded Lua HUD script
- Inputs: None
- Outputs/Return: `bool` (success/failure)
- Side effects: Executes Lua code; may trigger game state changes
- Calls: (not visible in this file)
- Notes: Must be called after `LoadLuaHUDScript`

### LoadHUDLua
- Signature: `void LoadHUDLua()`
- Purpose: Load HUD Lua script(s) from disk or default location
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes HUD Lua system; may load file from filesystem
- Calls: Likely calls `LoadLuaHUDScript` internally
- Notes: Higher-level convenience function

### CloseLuaHUDScript
- Signature: `void CloseLuaHUDScript()`
- Purpose: Unload and cleanup Lua HUD script
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates Lua state; calls `L_Call_HUDCleanup` if needed
- Calls: (not visible in this file)
- Notes: Inverse of `LoadHUDLua`

### MarkLuaHUDCollections
- Signature: `void MarkLuaHUDCollections(bool loading)`
- Purpose: Mark Lua HUD collections for garbage collection (loading phase)
- Inputs: `loading` (true during load, false during unload)
- Outputs/Return: None
- Side effects: Manages Lua GC roots for HUD state
- Calls: (not visible in this file)
- Notes: Called as part of Aleph One's collection lifecycle; enables safe GC during saves/loads

## Control Flow Notes
Fits into game startup (LoadHUDLua ΓåÆ LoadLuaHUDScript ΓåÆ RunLuaHUDScript), per-frame rendering (L_Call_HUDDraw), window resize handling (L_Call_HUDResize), and shutdown (CloseLuaHUDScript). The `bool loading` parameter in `MarkLuaHUDCollections` suggests integration with Aleph One's save/load cycle.

## External Dependencies
- `config.h` ΓÇô conditional compilation (`HAVE_LUA` guard)
- `cseries.h` ΓÇô base type definitions and macros (included indirectly)
- Lua library (linked externally; not included here)
