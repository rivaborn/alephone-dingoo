# Source_Files/GameWorld/marathon2.cpp

## File Purpose
Core game world orchestration for the Marathon engine, implementing the main game loop, world updates, and level lifecycle. Handles player action queue management and supports predictive client-side movement for networking.

## Core Responsibilities
- World initialization and shutdown
- Main game tick update loop with per-frame simulation
- Level loading/unloading and transitions
- Player action queue processing (real, Lua, and predictive)
- Predictive movement simulation for network lag compensation
- Polygon trigger activation (lights, platforms, monsters, items)
- Damage calculation and level completion state checking
- Game-over and level-change detection

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ModifiableActionQueues` | class | Queues for player action flags with player-per-channel organization |
| `player_data` | struct | Player state (from player.h) |
| `monster_data` | struct | Monster state (from monsters.h) |
| `object_data` | struct | Generic object state (from map.h) |
| `damage_definition` | struct | Damage parameters with type, base, random, scale (from map.h) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `GameQueue` | `ModifiableActionQueues*` | static | Intermediate action-flags queue; transfers flags from source to engine |
| `sMostRecentFlagsForPlayer[]` | `uint32[MAXIMUM_NUMBER_OF_PLAYERS]` | static | Caches most recent action flags per player for prediction |
| `sPredictionWanted` | `bool` | static | Flag: client-side movement prediction is active |
| `sPredictedTicks` | `size_t` | static | Count of predicted ticks awaiting real server confirmation |
| `sSavedPlayerData[]` | `player_data[MAXIMUM_NUMBER_OF_PLAYERS]` | static | Saved player state snapshot before entering predictive mode |
| `sSavedPlayerMonsterData[]` | `monster_data[MAXIMUM_NUMBER_OF_PLAYERS]` | static | Saved monster/object state for prediction rollback |
| `sSavedTickCount` | `int32` | static | Sanity check: tick count at prediction entry |
| `sSavedRandomSeed` | `uint16` | static | Sanity check: RNG seed at prediction entry |

## Key Functions / Methods

### initialize_marathon
- **Signature:** `void initialize_marathon(void)`
- **Purpose:** One-time engine initialization; allocates memory and initializes all game subsystems.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates memory for map, render, pathfinding, textures; initializes weapons, game window, scenery, items; creates `GameQueue` for action processing.
- **Calls:** `build_trig_tables()`, `allocate_map_memory()`, `allocate_pathfinding_memory()`, `allocate_texture_tables()`, `initialize_weapon_manager()`, `initialize_game_window()`, `initialize_scenery()`, `initialize_items()`, `OGL_Initialize()` (conditionally)
- **Notes:** Called once at startup. Conditional `OGL_Initialize()` only on OpenGL builds (not macOS).

### update_world
- **Signature:** `std::pair<bool, int16> update_world(void)`
- **Purpose:** Main game loop tick; advances simulation one or more ticks, processes action queues, handles prediction.
- **Inputs:** None (reads from global queues and network if networked)
- **Outputs/Return:** Pair of (whether_screen_changed, real_ticks_elapsed). Screen changes if prediction occurred or real ticks advanced.
- **Side effects:** Modifies `dynamic_world->tick_count`, updates all objects/players/monsters/effects, manages prediction state, updates UI fades.
- **Calls:** `NetProcessMessagesInGame()`, `overlay_queue_with_queue_into_queue()`, `update_world_elements_one_tick()`, `exit_predictive_mode()`, `enter_predictive_mode()`, `update_interface()`, `update_fades()`, `check_recording_replaying()`, `game_timed_out()`.
- **Notes:** Returns control even if no real ticks elapsed (prediction may have changed visuals). Speed-limited by heartbeat count or network time. Lua and real action queues overlay each other.

### update_world_elements_one_tick
- **Signature:** `static int update_world_elements_one_tick(void)`
- **Purpose:** Execute a single game tick: update all dynamic world elements.
- **Inputs:** None (reads `GameQueue`)
- **Outputs/Return:** Enum: `kUpdateNormalCompletion`, `kUpdateGameOver`, or `kUpdateChangeLevel`
- **Side effects:** Updates lights, media, platforms, players, projectiles, monsters, effects, scenery, items; increments tick count and decrements game time.
- **Calls:** `L_Call_Idle()`, `update_lights()`, `update_medias()`, `update_platforms()`, `update_control_panels()`, `update_players()`, `move_projectiles()`, `move_monsters()`, `update_effects()`, `recreate_objects()`, `handle_random_sound_image()`, `animate_scenery()`, `animate_items()`, `AnimTxtr_Update()`, `ChaseCam_Update()`, `update_net_game()`, `check_level_change()`, `game_is_over()`.
- **Notes:** Each call assumes exactly 1 tick (`?t==1`).

### enter_predictive_mode / exit_predictive_mode
- **Signature:** `static void enter_predictive_mode(void)` / `static void exit_predictive_mode(void)`
- **Purpose:** Save/restore partial game state to enable client-side movement prediction without polluting server state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Snapshots/restores player, monster, object, and parasitic object data; preserves interface flags during restore to avoid OOS on inventory scrolling.
- **Calls:** `get_player_data()`, `get_monster_data()`, `get_object_data()`, `remove_object_from_polygon_object_list()`, `deferred_add_object_to_polygon_object_list()`, `perform_deferred_polygon_object_list_manipulations()`, `get_random_seed()`.
- **Notes:** Assertions check monster/object indices match at exit. Defers object list re-insertion to rebuild exact pre-prediction state.

### overlay_queue_with_queue_into_queue
- **Signature:** `static bool overlay_queue_with_queue_into_queue(ActionQueues* inBaseQueues, ActionQueues* inOverlayQueues, ActionQueues* inOutputQueues)`
- **Purpose:** Merge action queues: overlay flags from `inOverlayQueues` onto `inBaseQueues` into `inOutputQueues`, one tick per player.
- **Inputs:** Base queue, overlay queue (may be NULL), output queue
- **Outputs/Return:** `true` if flags available for all players; `false` otherwise.
- **Side effects:** Dequeues from base and overlay; enqueues to output.
- **Calls:** `countActionFlags()`, `dequeueActionFlags()`, `enqueueActionFlags()`
- **Notes:** Always dequeues from base even if overridden; overlay (e.g., Lua) takes precedence if non-empty.

### entering_map / leaving_map
- **Signature:** `bool entering_map(bool restoring_saved)` / `void leaving_map(void)`
- **Purpose:** Level transitions. `entering_map` initializes a newly-loaded level; `leaving_map` cleans up the old level.
- **Inputs:** `entering_map`: whether restoring from save (skip Pfhortran init if true)
- **Outputs/Return:** `entering_map`: success status; `leaving_map`: none
- **Side effects:** Initializes/marks collections for loading/unloading, loads monster/game sounds, resets action queues, calls Lua init/cleanup, syncs network, handles weapon/environment checks, stops music/sounds.
- **Calls:** `initialize_monsters_for_new_level()`, `reset_paths()`, `mark_environment_collections()`, `load_collections()`, `load_all_monster_sounds()`, `load_all_game_sounds()`, `NetSync()`, `RunLuaScript()`, `CloseLuaScript()`, `L_Call_Init()`, `L_Call_Cleanup()`, `reset_action_queues()`, `stop_fade()`.
- **Notes:** `entering_map` returns false if any step fails, triggering `leaving_map()` for cleanup.

### changed_polygon
- **Signature:** `void changed_polygon(short original_polygon_index, short new_polygon_index, short player_index)`
- **Purpose:** Handle object entry into a new polygon; trigger lights, platforms, monsters, items.
- **Inputs:** Original and new polygon indices, player index (or NONE for non-player objects)
- **Outputs/Return:** None
- **Side effects:** Activates monsters, triggers items, toggles lights, changes platform state, marks must-explore polygons as normal.
- **Calls:** `get_polygon_data()`, `get_player_data()`, `activate_nearby_monsters()`, `trigger_nearby_items()`, `set_light_status()`, `platform_was_entered()`, `try_and_change_platform_state()`.
- **Notes:** Handles polygon types: monster triggers (visible/invisible/dual), light triggers, platform triggers, item triggers, must-explore, etc.

### calculate_level_completion_state
- **Signature:** `short calculate_level_completion_state(void)`
- **Purpose:** Check mission objectives and return completion state.
- **Inputs:** None (reads `static_world->mission_flags` and world state)
- **Outputs/Return:** `_level_unfinished`, `_level_finished`, or `_level_failed`
- **Side effects:** None
- **Calls:** `live_aliens_on_map()`, `unretrieved_items_on_map()`, `untoggled_repair_switches_on_level()`.
- **Notes:** Checks extermination, exploration, retrieval, repair, and rescue mission conditions. Rescue fails if >50% civilian casualties.

### calculate_damage
- **Signature:** `short calculate_damage(struct damage_definition *damage)`
- **Purpose:** Compute final damage with random variance and difficulty scaling.
- **Inputs:** Damage definition (type, base, random, scale, flags)
- **Outputs/Return:** Total damage (short)
- **Side effects:** None (pure computation)
- **Calls:** `global_random()`
- **Notes:** Alien damage is reduced at _wuss_level and _easy_level; higher difficulties receive no bonus.

### cause_polygon_damage
- **Signature:** `void cause_polygon_damage(short polygon_index, short monster_index)`
- **Purpose:** Apply periodic environmental damage (lava, goo) to a monster based on polygon type.
- **Inputs:** Polygon index, monster index
- **Outputs/Return:** None
- **Side effects:** Calls `damage_monster()` if tick count and altitude conditions are met.
- **Calls:** `get_polygon_data()`, `get_monster_data()`, `get_object_data()`, `damage_monster()`
- **Notes:** Uses frequency masks (MINOR_OUCH_FREQUENCY, MAJOR_OUCH_FREQUENCY) to throttle damage ticks. Only applies major damage unconditionally; minor requires object at floor height.

## Control Flow Notes
- **Initialization Phase:** `initialize_marathon()` runs once at startup.
- **Main Loop:** Each frame calls `update_world()`, which processes action queues and advances ticks until speed-limited.
- **Per-Tick:** `update_world_elements_one_tick()` cascades updates: lights ΓåÆ platforms ΓåÆ players ΓåÆ projectiles ΓåÆ monsters ΓåÆ effects, then UI.
- **Prediction:** If network-enabled and `sPredictionWanted`, a second pass predicts ahead using unconfirmed action flags, rollable via `enter/exit_predictive_mode()`.
- **Level Transitions:** `check_level_change()` triggers `changed_polygon()` or `calculate_level_completion_state()` ΓåÆ level change/game-over ΓåÆ `entering_map()` or `game_timed_out()`.

## External Dependencies
- **Core Game Subsystems:** `map.h` (geometry), `player.h` (player state), `monsters.h`, `projectiles.h`, `effects.h`, `weapons.h`, `items.h`, `platforms.h`, `lightsource.h`, `media.h` (fluids), `scenery.h`
- **Rendering & UI:** `render.h`, `interface.h`, `game_window.h`, `fades.h`
- **Audio:** `Music.h`, `SoundManager.h`
- **Network:** `network.h`, `network_games.h` (game sync, chat callbacks)
- **Scripting:** `lua_script.h`, `lua_hud_script.h` (Lua integration, action queue overlay)
- **Support:** `ChaseCam.h`, `OGL_Setup.h`, `AnimatedTextures.h`, `tags.h`, `Console.h`, `ActionQueues.h` (action flag queues), `Logging.h`

---

**Notes on Architecture:**
- The file is the primary orchestrator of game-world simulation, delegating subsystem updates to specialized modules.
- Prediction system decouples client-side responsiveness from server-authoritative state, critical for network play.
- Action queue overlay design permits Lua scripts and network events to override real input without reimplementing the update loop.
- Level transitions are asynchronous via state flags (`check_level_change()`) rather than exceptions, keeping simulation linear.
