# Source_Files/Network/network_games.h

## File Purpose
Header for network game management in Aleph One (Marathon engine). Declares functions for initializing networked multiplayer games, tracking player/team rankings and scores, generating UI text for scoreboards, and managing network-specific features like compass beacons.

## Core Responsibilities
- Initialize and update the state of network multiplayer games each frame
- Calculate and retrieve player and team rankings with kill/death statistics
- Format ranking and score information for in-game HUD and post-game screens
- Track direct player-on-player kills
- Determine game-over conditions and provide exit entry point flags
- Manage the network compass (beacon) system for player navigation
- Detect game-mode-specific features (scoring, ball/flag modes)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `player_ranking_data` | struct | Stores a player's index and numerical ranking; used to collect per-player score data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `team_netgame_parameters` | `int32[NUMBER_OF_TEAM_COLORS][2]` | global | Team-specific configuration for network games (meaning of second dimension not inferable) |

## Key Functions / Methods

### initialize_net_game
- Signature: `void initialize_net_game(void);`
- Purpose: Set up network game state at start
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies global/static network game state
- Calls: (not inferable)
- Notes: Likely called once per level/game start

### update_net_game
- Signature: `bool update_net_game(void);`
- Purpose: Advance network game state and check termination
- Inputs: None
- Outputs/Return: `true` if game is over, `false` otherwise
- Side effects: Updates global scoring, rankings, and game state
- Calls: (not inferable)
- Notes: Likely called once per frame

### get_player_net_ranking
- Signature: `int32 get_player_net_ranking(short player_index, short *kills, short *deaths, bool game_is_over);`
- Purpose: Retrieve a player's score and fill out kill/death counts
- Inputs: Player index, game-over status
- Outputs/Return: Ranking value; `kills` and `deaths` pointers filled with stats
- Side effects: Writes to output pointers
- Calls: (not inferable)
- Notes: Ranking meaning may vary by game type; stats likely differ when game is active vs. finished

### get_team_net_ranking
- Signature: `int32 get_team_net_ranking(short team, short *kills, short *deaths, bool game_is_over);`
- Purpose: Retrieve a team's score and aggregate kill/death counts
- Inputs: Team index, game-over status
- Outputs/Return: Ranking value; `kills` and `deaths` pointers filled
- Side effects: Writes to output pointers
- Calls: (not inferable)

### calculate_player_rankings
- Signature: `void calculate_player_rankings(struct player_ranking_data *rankings);`
- Purpose: Compute final player rankings (likely for post-game screen)
- Inputs: Array of `player_ranking_data` to populate
- Outputs/Return: Fills the rankings array
- Side effects: Writes to array
- Calls: (not inferable)

### calculate_ranking_text / calculate_ranking_text_for_post_game
- Signature: `void calculate_ranking_text(char *buffer, int32 ranking);` and `void calculate_ranking_text_for_post_game(char *buffer, int32 ranking);`
- Purpose: Format a ranking value as human-readable text (e.g., "10 pts", "1st place")
- Inputs: Text buffer, ranking integer
- Outputs/Return: Writes formatted string to buffer
- Side effects: Modifies buffer
- Calls: (not inferable)
- Notes: Two variants for in-game vs. post-game contexts

### game_is_over
- Signature: `bool game_is_over(void);`
- Purpose: Query overall game termination state
- Inputs: None
- Outputs/Return: `true` if game has ended
- Side effects: None
- Calls: (not inferable)

### get_network_compass_state
- Signature: `short get_network_compass_state(short player_index);`
- Purpose: Retrieve the compass/beacon visibility flags for a player
- Inputs: Player index
- Outputs/Return: Bit flags (see enum below)
- Side effects: None
- Calls: (not inferable)
- Notes: Flags control which compass quadrants (NW, NE, SW, SE) are active, plus beacon flag

**Other functions** (minor/self-explanatory): `current_net_game_has_scores()`, `current_game_has_balls()`, `player_killed_player()`, `get_network_joined_message()`, `get_entry_point_flags_for_game_type()`, `get_network_score_text_for_postgame()` ΓÇö manage game-type detection and per-game-mode features.

## Enums
- **Network compass state** (`_network_compass_*`): Bit flags for active compass directions and beacon; `0x000f` = all directions, `0x0010` = beacon enabled.

## Control Flow Notes
Typical frame-by-frame integration: `initialize_net_game()` runs once at game start; `update_net_game()` and ranking/compass queries occur each frame; post-game functions called when `game_is_over()` returns true.

## External Dependencies
- `config.h` ΓÇö feature detection (e.g., `HAVE_SDL_NET`, `DISABLE_NETWORKING` guard)
- Undefined constant `NUMBER_OF_TEAM_COLORS` (likely from game constants header)
- Other network/player/game state headers (not included here)
