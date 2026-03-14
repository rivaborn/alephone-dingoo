# Source_Files/GameWorld/player.cpp

## File Purpose
Core player entity implementation for the Marathon game engine. Manages player creation, state updates, damage/healing, weapon/powerup inventories, oxygen mechanics, death/respawn, and network synchronization across game ticks.

## Core Responsibilities
- **Player lifecycle**: Creation, initialization, deletion, and revival
- **Per-tick updates**: Physics integration, action processing, powerup decay, oxygen management
- **Damage system**: Calculate and apply damage, track damage given/taken per player/team
- **Powerup mechanics**: Invincibility, invisibility, infravision, extravision duration tracking
- **Oxygen/vacuum handling**: Depletion in vacuum, replenishment in air, suffocation deaths
- **Death management**: Kill player, handle reincarnation delays, manage dead player state
- **Serialization**: Pack/unpack player data for save games and network transmission
- **XML configuration**: Support MML-based configuration of initial items, damage responses, powerups, shapes

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `player_data` | struct | Complete player state: position, health, inventory, flags, physics variables |
| `damage_response_definition` | struct | Maps damage type ΓåÆ fade effect, sounds, death animation |
| `player_powerup_durations_definition` | struct | Configurable duration (ticks) for each powerup type |
| `player_settings_definition` | struct | Configurable player parameters: energy limits, oxygen rates, visual arc, vulnerability |
| `player_shape_definitions` | struct | Collection index and shape frame indices for all player animations |
| `player_powerup_definition` | struct | Maps powerup type ΓåÆ item ID for that powerup |
| `physics_variables` | struct | Player velocity, position, facing, elevation, step phase, floor/ceiling heights |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `players` | `player_data*` | global | Dynamic array of all active players |
| `local_player`, `current_player` | `player_data*` | global | Quick-access pointers to specific players |
| `local_player_index`, `current_player_index` | `short` | global | Indices of above |
| `team_damage_given`, `team_damage_taken` | `damage_record[8]` | global | Per-team cumulative damage statistics |
| `player_shapes` | `player_shape_definitions` | static | Shape frame indices for legs, torsos (idle/charging/firing), dying/dead states |
| `player_initial_items` | `short[]` | static | Starting inventory; settable via XML |
| `damage_response_definitions` | `damage_response_definition[]` | static | Per-damage-type fade/sound response; settable via XML |
| `player_powerup_durations` | `player_powerup_durations_definition` | static | Powerup duration values; settable via XML |
| `player_powerups` | `player_powerup_definition` | static | Item ID ΓåÆ powerup type mapping; settable via XML |
| `player_settings` | `player_settings_definition` | static | Energy, oxygen, vulnerability limits; settable via XML |
| `sRealActionQueues` | `ActionQueues*` | static | Global input queue for all players |
| `sLocalPlayerTicksSinceTerminal` | `int` | static | Tick counter tracking time in terminal mode |

## Key Functions / Methods

### new_player
- Signature: `short new_player(short team, short color, short identifier)`
- Purpose: Allocate and initialize a new player entity in the game world
- Inputs: Team color, player color, identifier (controls auto-recenter/weapon-switch flags)
- Outputs/Return: Player index in players array
- Side effects: Increments `dynamic_world->player_count`, allocates monster/object entities, initializes inventory
- Calls: `get_player_data`, `obj_clear`, `recreate_player`, `give_player_initial_items`, `try_and_strip_player_items`
- Notes: Assumes team/color are valid; inventory marked dirty for UI update

### update_players
- Signature: `void update_players(ActionQueues* inActionQueuesToUse, bool inPredictive)`
- Purpose: Main per-tick player state update; processes input, applies physics, manages powerups/oxygen
- Inputs: Action queue to read from, predictive flag (for network prediction)
- Outputs/Return: None (side effects only)
- Side effects: Modifies all player state; updates powerup durations, oxygen, weapon state, shape animations; handles death/respawn
- Calls: `dequeueActionFlags`, `update_player_physics_variables`, `handle_player_in_vacuum`, `ReplenishPlayerOxygen`, `update_player_weapons`, `update_action_key`, `update_player_teleport`, `update_player_media`, `set_player_shapes`
- Notes: Predictive mode skips powerup/oxygen decay and dead-player logic; handles ball-carrying run restriction, swimming, microphone arbitration, netdead detection

### damage_player
- Signature: `void damage_player(short monster_index, short aggressor_index, short aggressor_type, struct damage_definition *damage, short projectile_index)`
- Purpose: Apply damage to a player from an aggressor (monster or player), handle healing
- Inputs: Attacker monster index, attacker player index, damage definition
- Outputs/Return: None (state modified in-place)
- Side effects: Decreases suit_energy, triggers fade effect, plays damage sound, records damage given/taken, handles invincibility
- Calls: `get_damage_response`, `calculate_damage`, `play_object_sound`, `start_fade`, `record_damage_given`, `record_damage_taken`
- Notes: Negative damage heals; invincible players take damage from fusion only; damage clamped to [0, max_energy]

### kill_player
- Signature: `void kill_player(short player_index, short aggressor_player_index, short action)`
- Purpose: Kill a player and transition to death animations
- Inputs: Player to kill, aggressor (player that killed them), death action (_monster_is_dying_hard/soft)
- Outputs/Return: None
- Side effects: Sets player dead flags, starts death animation, places corpse shape, removes items, logs network disconnect if netdead
- Calls: `set_player_dead_shape`, `remove_dead_player_items`, `detonate_projectile` (for netdead)
- Notes: Dead hard = explosion; dead soft = fall; netdead kills are automatic

### revive_player
- Signature: `void revive_player(short player_index)`
- Purpose: Resurrect a dead player at a spawn point with full health
- Inputs: Player index
- Outputs/Return: None
- Side effects: Resets position to spawn, clears dead flags, refills health/oxygen, resets reincarnation delay, resets camera field of view
- Calls: `get_random_player_starting_location_and_facing`, `initialize_player_physics_variables`, `ResetFieldOfView`, `initialize_player_weapons`
- Notes: Called when reincarnation delay expires or single player revives manually

### pack_player_data / unpack_player_data
- Signature: `uint8* pack_player_data(uint8* Stream, player_data* Objects, size_t Count)` / `uint8* unpack_player_data(...)`
- Purpose: Serialize/deserialize player state for save games and network transmission
- Inputs: Stream pointer, player array, count
- Outputs/Return: Updated stream pointer
- Side effects: Writes/reads binary data, includes position, health, inventory, physics state, damage records, teleport state
- Calls: `ValueToStream`/`StreamToValue`, `ListToStream`/`StreamToList`, `PhysVarsToStream`/`StreamToPhysVars`
- Notes: Fixed 930-byte player_data size; includes per-player damage_taken array

## Control Flow Notes
**Init phase**: `initialize_players()` zeroes all player slots and action queues. `new_player()` creates individual players with initial items.

**Per-frame (update_players)**: 
1. Dequeue action flags for each player
2. Handle network dead detection
3. Skip updates for terminal-mode players
4. Apply action constraints (no run with ball, swimming toggles runΓåÆswim)
5. Update physics variables
6. Decay powerups and handle reincarnation delay
7. Update oxygen (deplete in vacuum, replenish in air)
8. Update weapons and map toggle
9. For dead players: check respawn conditions or increment reincarnation delay
10. For live players: pickup items, update weapons, check action key, handle damage polygons

**DeathΓåÆrespawn cycle**: Player dies ΓåÆ `kill_player()` sets dead flag ΓåÆ respawn delay begins ΓåÆ on action trigger + delay expired ΓåÆ `revive_player()` restores full state at spawn.

## External Dependencies
- **Map/World**: `map.h` (polygons, collision, damage polygons), `world.h` (3D points, vectors)
- **Monsters**: `monster_definitions.h`, `monsters.h` (used to treat player as monster for physics; guided missile activation)
- **Items/Weapons**: `items.h`, `weapons.h`, `projectiles.h` (player inventory, weapon state, initial items)
- **Network**: `network.h`, `network_games.h` (netdead detection, network parameters, team systems)
- **Audio**: `SoundManager.h` (damage/powerup/breathing sounds)
- **Input/Physics**: `ActionQueues.h`, `ChaseCam.h` (action queue system, camera)
- **Effects**: `fades.h` (damage fade effects), `effects.h` (impact effects referenced in damage definitions)
- **UI**: `interface.h`, `game_window.h`, `computer_interface.h` (terminal mode, screen output)
- **Serialization**: `Packing.h` (pack/unpack helper macros)
- **Scripting**: `lua_script.h` (Lua integration)
