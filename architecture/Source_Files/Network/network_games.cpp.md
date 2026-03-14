# Source_Files/Network/network_games.cpp

## File Purpose

Implements network multiplayer game-mode mechanics for the Aleph One engine (Marathon-compatible). Manages game-type-specific rules, scoring, and state for modes like King of the Hill, Capture the Flag, Rugby, Tag, and Defense. Provides ranking calculations, compass navigation beacons, and end-condition detection.

## Core Responsibilities

- Calculate player and team rankings based on game type and performance metrics
- Initialize game-specific state (beacon locations, ball spawning)
- Provide per-tick game updates (scoring, possession tracking, state transitions)
- Determine end-of-game conditions (kill limits, time limits)
- Track game-specific parameters (flag pulls, hill time, points scored, possession time)
- Generate formatted ranking text for HUD display (scores, percentages, time durations)
- Direct network compass beacons toward objectives (ball, it-player, hill)
- Handle ball possession and destruction in ball-based modes

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `player_data` | struct (external) | Player state; includes `netgame_parameters[2]` for mode-specific tracking |
| `polygon_data` | struct (external) | Map geometry; `type` field indicates hill/base/normal polygon |
| `game_data` | struct (external) | Game parameters; `kill_limit`, `game_type`, time remaining |
| `damage_record` | struct (external) | Damage dealt/taken; tracks kills and damage totals |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `team_netgame_parameters` | `int32[NUMBER_OF_TEAM_COLORS][2]` | static (file) | Per-team game metrics (one per index, e.g., flag pulls, time on hill) |

## Key Functions / Methods

### get_player_net_ranking
- **Signature:** `int32 get_player_net_ranking(short player_index, short *kills, short *deaths, bool game_is_over)`
- **Purpose:** Compute a player's score/ranking based on game type. Returns different metrics depending on mode (killsΓêÆdeaths, time on hill, flag pulls, etc.).
- **Inputs:** `player_index`, pointers to fill with kill/death counts, `game_is_over` flag
- **Outputs/Return:** Ranking value (int32); fills `*kills` and `*deaths`
- **Side effects:** Reads player data; may compute aggregate team damage
- **Calls:** `get_player_data()`, `GET_GAME_TYPE()` macro, `GetLuaScoringMode()` (if Lua enabled)
- **Notes:** Branches heavily on game type. For defense mode, compares offender time vs. defending team time. Handles Lua custom scoring modes.

### get_team_net_ranking
- **Signature:** `int32 get_team_net_ranking(short team, short *kills, short *deaths, bool game_is_over)`
- **Purpose:** Compute team ranking; parallels player ranking but aggregates across team members.
- **Inputs:** `team` index, pointers for kill/death counts
- **Outputs/Return:** Ranking value; fills kill/death counts
- **Side effects:** Reads team damage records
- **Calls:** `GET_GAME_TYPE()`, `GetLuaScoringMode()`
- **Notes:** Iterates team damage arrays and `team_netgame_parameters`.

### initialize_net_game
- **Signature:** `void initialize_net_game(void)`
- **Purpose:** Initialize game state when network game starts. Clears team parameters; sets up mode-specific beacons (hill center for KOTH/Defense, none for others).
- **Inputs:** None (reads `GET_GAME_TYPE()`, `dynamic_world`)
- **Outputs/Return:** None (modifies `dynamic_world->game_beacon`, `dynamic_world->game_player_index`)
- **Side effects:** Sets global game state; calculates polygon center for hill-based modes
- **Calls:** `GET_GAME_TYPE()`, iterates `map_polygons`, `get_polygon_data()`
- **Notes:** For King of the Hill and Defense, computes beacon as average of all hill-type polygons. Initializes `game_player_index` to `NONE` for ball/tag modes.

### get_network_compass_state
- **Signature:** `short get_network_compass_state(short player_index)`
- **Purpose:** Determine compass display state (direction bits toward objective: NE, NW, SE, SW) based on game type and player status. Supports Lua-scriptable compass.
- **Inputs:** `player_index`
- **Outputs/Return:** Bitmap of compass directions (e.g., `_network_compass_ne | _network_compass_se`)
- **Side effects:** Reads player location, facing, beacon location
- **Calls:** `GET_GAME_TYPE()`, `get_player_data()`, `get_polygon_data()`, `get_object_data()`, `arctangent()`
- **Notes:** Checks player's supporting polygon type to determine if at objective (on hill ΓåÆ all-on). Otherwise, beams to objective player/ball location. KOTH/Defense/Rugby/KTMWTB all support beacons. Tag mode points to "it" player.

### player_killed_player
- **Signature:** `bool player_killed_player(short dead_player_index, short aggressor_player_index)`
- **Purpose:** Handle kill attribution; determine if kill should count. Special logic for tag mode (transfer "it" status).
- **Inputs:** Dead and aggressor player indices
- **Outputs/Return:** Boolean (true = count kill; false = don't attribute)
- **Side effects:** May update `dynamic_world->game_player_index` (tag mode "it" transfer)
- **Calls:** `GET_GAME_TYPE()`, `mark_player_network_stats_as_dirty()`
- **Notes:** Tag mode logic: if aggressor is "it" or dead player kills self, "it" transfers to dead player (unless already none). Multi-player check prevents single-player game logic.

### update_net_game
- **Signature:** `bool update_net_game(void)`
- **Purpose:** Per-tick network game update. Increments game-specific counters (time on hill, possession time) and checks scoring events (flag captures, goals). Returns false (game-over return ignored per comment).
- **Inputs:** None (reads global `dynamic_world`, player/polygon data)
- **Outputs/Return:** Boolean (always false; see comment re: `game_is_over()`)
- **Side effects:** Modifies `player->netgame_parameters`, `team_netgame_parameters`, calls `destroy_players_ball()`, decrements `current_player->interface_decay`
- **Calls:** `GET_GAME_TYPE()`, `get_player_data()`, `get_polygon_data()`, `player_has_ball()`, `find_player_ball_color()`, `destroy_players_ball()`, `PLAYER_IS_DEAD()` macro
- **Notes:** Captures (CTF): checks if player in own base with enemy flag; rugby: checks if player in enemy base or hill with ball (scores goal, destroys ball). Defense mode: increments offender base time only for first offender found. Interface decay marks stats dirty for HUD refresh.

### calculate_player_rankings
- **Signature:** `void calculate_player_rankings(struct player_ranking_data *rankings)`
- **Purpose:** Sort players by ranking into output array (highest first).
- **Inputs:** Output array pointer
- **Outputs/Return:** Fills `rankings` array
- **Side effects:** None (reads only)
- **Calls:** `get_player_net_ranking()` repeatedly
- **Notes:** Bubble-sort variant; O(n┬▓). Temporary copy used to avoid in-place sort corruption.

### calculate_ranking_text
- **Signature:** `void calculate_ranking_text(char *buffer, int32 ranking)`
- **Purpose:** Format ranking value as string suitable for HUD display (e.g., "5" for kills, "1:23" for time).
- **Inputs:** `ranking` value; game type determined via macro
- **Outputs/Return:** Fills `buffer` with formatted text (e.g., "5:30" for time mode)
- **Side effects:** Writes to buffer
- **Calls:** `GET_GAME_TYPE()`, `GetLuaScoringMode()`, `sprintf()`
- **Notes:** Mode-dependent formatting (percentages for coop, time MM:SS for hill/tag/KTMWTB/defense, plain numbers otherwise).

### game_is_over
- **Signature:** `bool game_is_over(void)`
- **Purpose:** Determine if game-end condition is met (time expired, kill/flag limit reached, or Lua end condition).
- **Inputs:** None (reads global state)
- **Outputs/Return:** Boolean
- **Side effects:** May disable kill-limit flag and set 2-second countdown on meeting limit (to let player death animate)
- **Calls:** `GET_GAME_TYPE()`, `GET_GAME_OPTIONS()`, `GetLuaGameEndCondition()`, `get_player_data()`, iterates players/teams
- **Notes:** Kill-limit check varies per mode: player kills for carnage/king/tag, team flag pulls for CTF, team points for rugby, offender time for defense. Lua can override with `_game_end_now_condition` or `_game_no_end_condition`.

### current_net_game_has_scores
- **Signature:** `bool current_net_game_has_scores(void)`
- **Purpose:** Query whether current game mode displays/tracks scores (vs. pure deathmatch).
- **Inputs:** None
- **Outputs/Return:** Boolean (true for CTF, flag pulls, etc.; false for carnage, coop)
- **Side effects:** None
- **Calls:** `GET_GAME_TYPE()`

### current_game_has_balls
- **Signature:** `bool current_game_has_balls(void)`
- **Purpose:** Determine if game mode uses ball items (KTMWTB, CTF, Rugby).
- **Inputs:** None
- **Outputs/Return:** Boolean
- **Side effects:** None
- **Calls:** `GET_GAME_TYPE()`

## Control Flow Notes

**Initialization Phase:** `initialize_net_game()` called on map load; sets up beacons and clears team parameters.

**Frame Update:** `update_net_game()` called each tick; increments mode-specific counters and handles scoring events (captures, goals).

**End Condition Check:** `game_is_over()` polled each frame to detect time/limit expiration and trigger 2-second graceful shutdown.

**Ranking Display:** `calculate_player_rankings()` and `calculate_ranking_text()` called by UI/HUD code to generate leaderboard and stat strings.

**Network Compass:** `get_network_compass_state()` called each frame per player to direct compass toward objective.

## External Dependencies

- **map.h:** `polygon_data`, `GET_GAME_TYPE()`, `GET_GAME_OPTIONS()` macros, `dynamic_world`, `map_polygons`, game type enums, `TICKS_PER_SECOND`
- **player.h:** `player_data`, `get_player_data()`, `PLAYER_IS_DEAD()` macro, team color enums
- **items.h:** `find_player_ball_color()`, `BALL_ITEM_BASE`
- **network.h:** Network game declarations (interface only)
- **lua_script.h:** `GetLuaScoringMode()`, `GetLuaGameEndCondition()` (conditional: `HAVE_LUA`)
- **game_window.h:** `mark_player_network_stats_as_dirty()`
- **SoundManager.h:** Sound definitions (not actively used in this file)
- **weapons.c/external:** `destroy_players_ball()` (declaration; definition in weapons.c)

**Defined Elsewhere:** `arctangent()`, `csprintf()`, `vhalt()`, `getcstr()`, `NORMALIZE_ANGLE()`, `QUARTER_CIRCLE`, `HALF_CIRCLE`, `FULL_CIRCLE`, macros for object/player access.
