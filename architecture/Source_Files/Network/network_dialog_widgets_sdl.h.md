# Source_Files/Network/network_dialog_widgets_sdl.h

## File Purpose
Defines three custom SDL widget classes for network-related UI dialogs: discovering network players, displaying in-game player status, and selecting map entry points based on game type. Supports both gather/join dialogs and postgame carnage reports.

## Core Responsibilities
- Manage and display lists of remote players discovered via SSLP network protocol
- Render player icons, names, and kill/score statistics in game lobbies and postgame reports
- Validate and present playable entry points (levels) filtered by selected game type
- Delegate user interactions (clicks, selections) to callback handlers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `player_entry2` | struct | Stores display info for one player: name, color, icon width, and rendered image |
| `prospective_joiner_info` | struct (defined elsewhere) | Network player discovered via SSLP |
| `entry_point` | struct (from map.h) | Game level entry point with metadata |
| `net_rank` | struct (from network_dialogs.h) | Player ranking data: kills, deaths, scores |

## Global / File-Static State
None.

## Key Functions / Methods

### w_found_players (inherits from `w_list<prospective_joiner_info>`)
Manages the list of players discovered on the network for the Gather dialog.

#### found_player
- **Purpose:** Register discovery of a new remote player
- **Inputs:** `prospective_joiner_info &player` ΓÇô discovered player record
- **Side effects:** Adds to `found_players` and `listed_players` vectors; marks for redraw

#### update_player
- **Purpose:** Refresh entry for an already-known player
- **Inputs:** `prospective_joiner_info &player` ΓÇô updated record
- **Side effects:** Updates corresponding entry in vectors

#### hide_player
- **Purpose:** Remove player from display without forgetting it
- **Inputs:** `prospective_joiner_info &player` ΓÇô player to hide
- **Side effects:** Adds to `hidden_players` vector; updates `listed_players`

#### item_selected
- **Purpose:** Invoked when user clicks on a player in the list
- **Side effects:** Calls `player_selected_callback` if set

#### callback_on_all_items
- **Purpose:** Iterate all items and invoke callback for each
- **Notes:** Comment states callback is expected to remove the item; used for cleanup

#### draw_item
- **Purpose:** Render a single player entry in the list widget
- **Inputs:** Iterator to player, SDL surface, x/y position, width, selected flag
- **Notes:** Called by list base class during draw phase

### w_players_in_game2 (inherits from `widget`)
Dual-purpose widget: displays players in gather/join dialogs AND renders postgame carnage reports.

#### Constructor
- **Signature:** `w_players_in_game2(bool inPostgameLayout)`
- **Purpose:** Initialize with layout suitable for pregame or postgame display
- **Inputs:** `inPostgameLayout` ΓÇô true for larger postgame report, false for compact pregame display

#### update_display
- **Purpose:** Sync rendering data from game world state or topology
- **Inputs:** `inFromDynamicWorld` ΓÇô if true, read from dynamic world; else from static topology
- **Side effects:** Rebuilds `player_entries` vector; marks widget dirty

#### start_displaying_actual_information
- **Purpose:** Signal that topology/world data is now valid (used to suppress placeholder rendering)
- **Side effects:** Sets `displaying_actual_information = true`

#### draw
- **Purpose:** Render the widget (player icons, names, kill/score bars, legend, totals for postgame)
- **Inputs:** `SDL_Surface *s` ΓÇô target surface
- **Outputs/Return:** Void; draws directly to surface
- **Side effects:** Calls helper draw methods based on layout mode (separate vs. clumped by team, carnage vs. scores)

#### click
- **Purpose:** Handle mouse click; invoke callback if click is near a player icon
- **Inputs:** x, y coordinates relative to widget
- **Side effects:** Calls `element_clicked_callback` if set and click is valid

#### set_graph_data
- **Purpose:** Provide player ranking and scoring data for postgame carnage report rendering
- **Inputs:** Pointer to `net_rank` array, count, selected player index, team-clumping flag, carnage-vs-scores flag
- **Side effects:** Populates internal ranking structures; marks widget dirty

#### Helper draw methods
- `draw_player_icon(...)` ΓÇô render single player's faction/color icon at given position
- `draw_player_icons_separately(...)` / `draw_player_icons_clumped(...)` ΓÇô render all icons in row or grouped by team
- `draw_player_names_separately(...)` / `draw_player_names_clumped(...)` ΓÇô render player names with text layout helper
- `draw_bar_or_bars(...)` ΓÇô render kill/score bar(s) for one ranking position; returns bar metrics
- `draw_bars_separately(...)` / `draw_bars_clumped(...)` ΓÇô render all bars (carnage or scores)
- `draw_bar(...)` ΓÇô render single colored bar with value, return bounding rect
- `draw_bar_labels(...)` ΓÇô label bars with player/team names
- `draw_carnage_totals(...)` ΓÇô render team kill/death totals
- `draw_carnage_legend(...)` ΓÇô render color key for kill/death/suicide/friendly-fire

### w_entry_point_selector (inherits from `w_select_button`)
Dropdown selector for choosing a playable level, filtered by game type and map file.

#### Constructor
- **Signature:** `w_entry_point_selector(size_t inGameType, int16 inLevelNumber)`
- **Purpose:** Initialize selector for given game type and initial level
- **Side effects:** Calls `validateEntryPoint()` to ensure initial level is valid for game type

#### setGameType
- **Purpose:** Change game type; adjust entry point if current level is incompatible
- **Inputs:** `inGameType` ΓÇô new game type
- **Side effects:** Revalidates entry point; marks widget dirty if changed

#### reset
- **Purpose:** Choose first available entry point (useful when map file changes)
- **Side effects:** Sets level to NONE, calls `validateEntryPoint()`

#### setLevelNumber
- **Purpose:** Request specific level by number
- **Inputs:** `inLevelNumber` ΓÇô desired level number
- **Side effects:** Calls `validateEntryPoint()` to resolve

#### getEntryPoint
- **Purpose:** Return the currently selected entry point
- **Outputs/Return:** `const entry_point&` ΓÇô current selection

#### event
- **Purpose:** Handle user input; support cursor left/right to cycle through levels
- **Inputs:** `SDL_Event& e`
- **Side effects:** Updates selection; may mark dirty

#### validateEntryPoint
- **Purpose:** Look up entry point by level number and game type from active map file
- **Side effects:** Sets `mEntryPoint` and `mCurrentIndex` based on lookup; falls back to first available if requested level doesn't support game type; sets level to NONE if no valid entry points exist

## Control Flow Notes
- **Pregame phase:** `w_found_players` accumulates remote players via SSLP callbacks; `w_players_in_game2` displays joined participants; `w_entry_point_selector` lets host choose level.
- **Postgame phase:** `w_players_in_game2` switches to carnage-report mode and renders kill/death statistics from `net_rank` array.
- All three widgets inherit from SDL widget base and participate in dialog event dispatch and rendering.

## External Dependencies
- **sdl_widgets.h** ΓÇô `w_list<T>` template, `w_select_button`, `widget` base class
- **SSLP_API.h** ΓÇô `prospective_joiner_info` struct for network-discovered players
- **player.h** ΓÇô `MAXIMUM_PLAYER_NAME_LENGTH`, team colors, player constants
- **PlayerImage_sdl.h** ΓÇô `PlayerImage` class for rendering player icons
- **network_dialogs.h** ΓÇô `net_rank` struct, graphics callback typedefs
- SDL library ΓÇô `SDL_Surface`, `SDL_Event`
- Defined elsewhere: `TextLayoutHelper`, `bar_info` (forward declared)

**Compile guard:** `#if !defined(DISABLE_NETWORKING)` ΓÇô entire file omitted if networking disabled.
