# Source_Files/Network/network_dialog_widgets_sdl.cpp

## File Purpose
Implements SDL-based dialog widgets for network features in Aleph One: player discovery/listing, in-game player display with postgame carnage reports, and level entry-point selection. These widgets integrate with the game's networking subsystem and HUD rendering.

## Core Responsibilities
- `w_found_players`: Widget for listing discovered network players; manages found/hidden/listed player states and callbacks
- `w_players_in_game2`: Dual-purpose widget for displaying active players during gather/join AND as postgame carnage-report graph
- `w_entry_point_selector`: Button widget for choosing level entry points from current map filtered by game type
- Player display: Renders player icons (3D images), names with overlap-avoidance, team/color information
- Carnage statistics: Draws score or kill/death bars, totals, and legends with spatial layout management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `prospective_joiner_info` | struct | Network player data: stream_id, name, color, team, gathering status |
| `player_entry2` | struct | Display info for a player: name, pixel color, width, 3D PlayerImage |
| `bar_info` | struct | Metadata for drawing score/kill bars: center_x, top_y, pixel_color, label_text |
| `w_found_players` | class | List widget for discovered players; extends `w_list<prospective_joiner_info>` |
| `w_players_in_game2` | class | Message widget for in-game/postgame player display; extends `widget` |
| `w_entry_point_selector` | class | Select-button widget for level entry points; extends `w_select_button` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `operator==` | function | global | Equality for `prospective_joiner_info` (compares stream_id); used by `find_item_index_in_vector` |
| `find_item_index_in_vector<T>` | template function | file-static | Linear search for item in vector; returns size_t (-1 if not found) |
| Layout enums | constants | file-static | `kWPIG2Width`, `kMaxHeadroom`, `kNameOffset`, `kBarWidth`, etc. for positioning calculations |

## Key Functions / Methods

### w_found_players::found_player
- **Signature:** `void found_player(prospective_joiner_info &player)`
- **Purpose:** Register a newly discovered network player.
- **Inputs:** Player info struct with network identity and metadata.
- **Outputs/Return:** None; updates internal `found_players` vector.
- **Side effects:** Calls `list_player()` to add to display list; marks widget dirty.
- **Calls:** `list_player()`
- **Notes:** Assumes external code will manage duplicates.

### w_found_players::unlist_player
- **Signature:** `void unlist_player(const prospective_joiner_info &player)`
- **Purpose:** Remove a player from the displayed list (e.g., left game).
- **Inputs:** Player info to remove.
- **Outputs/Return:** None; modifies `listed_players` and scroll state.
- **Side effects:** Adjusts scroll position if removed item was at/above top; recomputes list height.
- **Calls:** `find_item_index_in_vector()`, `set_top_item()`
- **Notes:** No-op if player not in list; preserves scroll position when possible.

### w_players_in_game2::update_display
- **Signature:** `void update_display(bool inFromDynamicWorld = false)`
- **Purpose:** Populate display from current player data (network topology or dynamic world).
- **Inputs:** Boolean flag: true for postgame (dynamic_world), false for in-game network data.
- **Outputs/Return:** None; rebuilds internal `player_entries` and `players_on_team` vectors.
- **Side effects:** Creates new `PlayerImage` objects; clears prior entries. Marks widget dirty.
- **Calls:** `NetGetNumberOfPlayers()`, `NetGetPlayerData()`, `get_player_data()`, `text_width()`, `get_dialog_player_color()`, `PlayerImage` constructor.
- **Notes:** Runs at gather/join or postgame; uses conditional logic for data source.

### w_players_in_game2::draw
- **Signature:** `void draw(SDL_Surface* s) const`
- **Purpose:** Render all player icons, names, bars, and legend (if postgame graph enabled).
- **Inputs:** SDL surface to draw on.
- **Outputs/Return:** None; side effect is surface rendering.
- **Side effects:** Sets/resets clip rectangle; modifies `TextLayoutHelper` to avoid text overlap.
- **Calls:** `set_drawing_clip_rectangle()`, `draw_player_icons_*()`, `draw_bars_*()`, `draw_player_names_*()`, `draw_carnage_totals()`, `draw_carnage_legend()`, `draw_bar_labels()`.
- **Notes:** Order is back-to-front: icons ΓåÆ bars ΓåÆ names ΓåÆ totals/legend ΓåÆ labels. Two layout modes: clumped by team or separate.

### w_players_in_game2::draw_bar
- **Signature:** `void draw_bar(SDL_Surface* s, int inCenterX, int inBarColorIndex, int inBarValue, int inMaxValue, bar_info& outBarInfo) const`
- **Purpose:** Draw a single score or kill/death bar with 3D bevel effect.
- **Inputs:** Center X, bar color index, value to draw, max value for scaling, output bar metadata.
- **Outputs/Return:** Populates `outBarInfo` (center_x, top_y, pixel_color).
- **Side effects:** Draws filled rectangles on surface; calls `SDL_FillRect()` three times (light, dark, middle).
- **Calls:** `get_net_color()`, `SDL_MapRGB()`, `SDL_FillRect()`.
- **Notes:** Handles negative values (e.g., Tag mode); scales proportionally to max. Draws dark bevel on right/top, lighter fill in middle.

### w_entry_point_selector::validateEntryPoint
- **Signature:** `void validateEntryPoint()`
- **Purpose:** Ensure current entry point is valid for game type; select first valid if not.
- **Inputs:** Uses member `mGameType` and `mEntryPoint.level_number`.
- **Outputs/Return:** None; updates `mEntryPoint`, `mCurrentIndex`, marks dirty.
- **Side effects:** Queries entry points from map; may reset level to first available.
- **Calls:** `get_entry_point_flags_for_game_type()`, `get_entry_points()`.
- **Notes:** Called on game-type or map change; preserves selection if valid, otherwise reverts to index 0.

### w_entry_point_selector::event
- **Signature:** `virtual void event(SDL_Event &e)`
- **Purpose:** Handle keyboard navigation (left/right arrows to cycle entry points).
- **Inputs:** SDL_Event (checked for SDLK_LEFT/SDLK_RIGHT).
- **Outputs/Return:** None; modifies event type to SDL_NOEVENT if consumed.
- **Side effects:** Updates `mCurrentIndex` cyclically; calls `play_dialog_sound()`.
- **Calls:** `play_dialog_sound()`.
- **Notes:** Only active if more than one entry point; swallows event after handling.

## Control Flow Notes
- **Initialization:** Widgets created during dialog setup; `update_display()` called to populate from live data.
- **Gather/Join phase:** `w_found_players` and `w_entry_point_selector` active; `w_players_in_game2` shows current roster.
- **Postgame phase:** `w_players_in_game2` re-rendered with carnage data via `set_graph_data()`.
- **Rendering:** `draw()` called per frame; uses clip rectangles to constrain output.
- **Events:** `click()` and `event()` handle user interaction; callbacks notify application of selections.

## External Dependencies
- **screen_drawing.h**: `draw_text()`, `text_width()`, `set_drawing_clip_rectangle()`, `get_theme_color()`, `get_theme_font()`, color/font access.
- **sdl_fonts.h**: `font_info` class for text measurement and drawing.
- **interface.h**: `get_dialog_player_color()`, `clear_screen()`, theme utilities.
- **network.h**: `NetGetNumberOfPlayers()`, `NetGetPlayerData()`, prospective joiner info.
- **player.h**: `player_data` struct, `get_player_data()`, player constants.
- **HUDRenderer.h**: `PlayerImage` class for rendering 3D player models.
- **shell.h**, **collection_definition.h**: For shape/collection management.
- **preferences.h**, **screen.h**: Environment and map file access.
- **TextStrings.h**: `TS_GetCString()` for localized UI text.
- **TextLayoutHelper.h**: Text layout helper for overlap avoidance.
- **SDL** (via screen_drawing.h): `SDL_Surface`, `SDL_Rect`, `SDL_FillRect()`, `SDL_MapRGB()`.
