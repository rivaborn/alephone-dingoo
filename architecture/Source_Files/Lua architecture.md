# Subsystem Overview

## Purpose
Lua scripting subsystem provides an embedded scripting interface enabling level designers and modders to customize gameplay behavior through Lua scripts. Exposes game state (players, monsters, objects, map geometry) and mechanics to Lua code via C API bindings, dispatches game events to script callbacks, and manages script lifecycle and persistence.

## Key Files
| File | Role |
|------|------|
| lua.h | Public C API header; state lifecycle, stack operations, execution control |
| lua_script.h / lua_script.cpp | Main script event dispatcher; lifecycle management; state serialization |
| lua_hud_script.h / lua_hud_script.cpp | HUD script loader and lifecycle (init, draw, resize, cleanup) |
| lua_templates.h | C++ template metaprogramming for binding C++ objects and enums to Lua |
| lua_hud_objects.h / lua_hud_objects.cpp | Lua bindings for HUD state (screen, player, game, renderer) |
| lua_map.h / lua_map.cpp | Lua bindings for map geometry (polygons, lines, sides, lights, platforms) |
| lua_monsters.h / lua_monsters.cpp | Lua bindings for monster objects, types, and actions |
| lua_objects.h / lua_objects.cpp | Lua bindings for world objects (effects, items, scenery, sounds) |
| lua_player.h / lua_player.cpp | Lua bindings for player state, camera, inventory, weapons |
| lua_projectiles.h / lua_projectiles.cpp | Lua bindings for projectile objects and types |
| lua_serialize.h / lua_serialize.cpp | Binary serialization/deserialization of Lua values |
| lapi.h, lauxlib.h, lcode.h, ldebug.h, ldo.h, lfunc.h, lgc.h, llex.h, llimits.h, lmem.h, lobject.h, lopcodes.h, lparser.h, lstate.h, lstring.h, ltable.h, ltm.h, lvm.h, lzio.h | Lua 5.1 VM internals (execution, memory, bytecode, garbage collection) |
| lualib.h, lundump.h, luaconf.h | Lua standard library and configuration |
| lua_mnemonics.h, language_definition.h | String-to-integer mnemonic mappings for game entities |

## Core Responsibilities
- Load Lua scripts from buffers and manage VM lifecycle (creation, execution, cleanup)
- Dispatch game events to Lua trigger callbacks (player killed, item picked, platform switched, damage taken)
- Register C++ functions and objects as Lua methods and classes via template-based bindings
- Expose game entity properties (position, type, health, state) as Lua-accessible fields with getter/setter methods
- Enable object creation and manipulation from Lua (spawn effects, items, monsters; modify health/position)
- Provide string mnemonics for game constants (item names, monster types, damage sources, game modes)
- Serialize and deserialize Lua table state for save/load and level transitions
- Maintain per-script global state (compass override, weapon selection, scoring mode)
- Support conditional I/O access (solo vs. multiplayer script modes) and script muting
- Implement compatibility layer mapping legacy Lua API calls to new bindings

## Key Interfaces & Data Flow
**Exposes to game engine:**
- Script lifecycle functions (`LoadLuaHUDScript`, `RunLuaHUDScript`, `InitializeLuaScript`, `CloseLuaScript`)
- Event dispatch functions (called from game systems: `lua_player_died`, `lua_item_created`, `lua_platform_switch`, `lua_damage_taken`)
- Queries for Lua-controlled state (scoring mode, weapon availability, camera paths, texture overrides)
- Camera/cutscene path system with timed keyframes

**Consumes from other subsystems:**
- Player state (`player.h`, `ActionQueues.h`) ΓÇö position, weapons, energy, inventory
- Monster data (`monsters.h`) ΓÇö type, position, health, behavior
- Game world (`map.h`, `world.h`) ΓÇö geometry, platforms, lights, media
- Sound system (`SoundManager.h`) ΓÇö sound registration and playback
- Rendering (`HUDRenderer.h`, `render.h`, `screen.h`) ΓÇö HUD drawing, screen properties
- Game physics and mechanics (projectiles, effects, items, scenery)
- Game mode state (difficulty, game type, team data)

## Runtime Role
- **Initialization**: `LuaHUDState` singleton created at engine startup; HUD script loaded from user preferences; script state initialized via `init` callback
- **Per-frame event dispatch**: Game systems call Lua trigger functions during gameplay (`Update`, `Draw` callbacks for HUD; event callbacks for game state changes)
- **Entity invalidation**: Signals sent to Lua when monsters, projectiles, or objects are destroyed to prevent stale references
- **Shutdown**: Lua state cleaned up when scripts are closed or engine exits; script state serialized for save files

## Notable Implementation Details
- Entire subsystem gated behind `HAVE_LUA` preprocessor conditional; can be compiled out for minimal builds
- Template-based binding system (`L_Class<>`, `L_Enum<>`, `L_Container<>`) abstracts C++/Lua object marshaling; reduces boilerplate for exposing game types
- Multiple script instances supported: embedded (level scripts), network (multiplayer), solo (offline); conditional I/O access per instance
- Version-controlled binary serialization format with reference tracking for shared/cyclic structures; enables save/load persistence
- Compatibility layer loads embedded Lua code mapping old-style function names to new template-based API (compatibility shims)
- Stack-based value exchange via Lua C API; no direct C++ object pointers exposed to Lua (indices used instead for safety)
