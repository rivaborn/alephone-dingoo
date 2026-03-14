# Source_Files/RenderOther/HUDRenderer.cpp

## File Purpose
Core implementation of HUD rendering for the Marathon-like game engine (Aleph One). Manages all on-screen UI elements including energy/oxygen bars, weapon and ammunition displays, inventory panels, and network player information. Supports both standard HUD and a Lua-driven texture palette viewer for debugging/asset previewing.

## Core Responsibilities
- Update all HUD elements conditionally based on dirty flags (energy, oxygen, weapon, ammo, inventory)
- Render progress bars (shield/energy and oxygen) with multi-level states
- Display weapon panel with single/multi-weapon variations and naming
- Render ammunition counts as grid layouts or energy bars
- Manage inventory display with category headers, item lists, and validity indicators
- Draw network-mode overlays (player rankings, game timer, kill counter)
- Support optional Lua texture palette viewer (gp2x/dingoo platform hack)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `weapon_interface_ammo_data` | struct | Ammo display layout (position, grid dimensions, bullet shapes) |
| `weapon_interface_data` | struct | Weapon panel config (shapes, text bounds, multi-weapon support) |
| `interface_state_data` | struct | Dirty flags tracking which HUD elements need redraw |
| `screen_rectangle` | struct | Screen coordinates (left/top/right/bottom) for UI placement |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `delay_time` | short | static (update_suit_oxygen) | Throttles oxygen bar redraws to 2 Hz |
| `ForceUpdate` | bool | member (HUD_Class) | Signals if any HUD element was redrawn this frame |
| `interface_state` | interface_state_data | extern | Tracks ammo/weapon/shield/oxygen dirty flags |
| `weapon_interface_definitions` | array | extern | 10-element array of weapon panel configurations |
| `current_player` | player_data* | extern | Pointer to active player (suit energy, oxygen, inventory) |
| `current_player_index` | short | extern | Index of active player in multiplayer |
| `dynamic_world` | world_data* | extern | Game world state (player count, game info, rankings) |

## Key Functions / Methods

### update_everything
- **Signature:** `bool HUD_Class::update_everything(short time_elapsed)`
- **Purpose:** Main frame update entry point; conditionally redraws all HUD elements.
- **Inputs:** `time_elapsed` ΓÇö elapsed ms since last frame (NONE=force redraw)
- **Outputs/Return:** `ForceUpdate` flag (true if any element was redrawn)
- **Side effects:** Sets dirty flags to false; may set `ForceUpdate=true`; calls all update_* methods
- **Calls:** `update_motion_sensor`, `update_inventory_panel`, `update_weapon_panel`, `update_ammo_display`, `update_suit_energy`, `update_suit_oxygen`, `draw_message_area`, plus Lua palette functions
- **Notes:** Lua texture palette viewer takes precedence; locks out standard HUD if active. Multiplayer messaging is drawn only if player_count > 1.

### update_suit_energy
- **Signature:** `void HUD_Class::update_suit_energy(short time_elapsed)`
- **Purpose:** Renders shield/energy bar with 1ΓÇô3-level appearance based on suit_energy value.
- **Inputs:** `time_elapsed` (NONE = force redraw)
- **Outputs/Return:** None (calls `draw_bar`)
- **Side effects:** Sets `shield_is_dirty=false`; may set `ForceUpdate=true`
- **Calls:** `get_interface_rectangle`, `draw_bar`, `BUILD_DESCRIPTOR`
- **Notes:** Supports 3 energy tiers (empty, single, double, triple bars). Normalizes overflow energy (modulo PLAYER_MAXIMUM_SUIT_ENERGY).

### update_suit_oxygen
- **Signature:** `void HUD_Class::update_suit_oxygen(short time_elapsed)`
- **Purpose:** Renders oxygen bar with 2-tick throttling to reduce CPU load.
- **Inputs:** `time_elapsed`
- **Outputs/Return:** None
- **Side effects:** Decrements static `delay_time`; resets it to DELAY_TICKS_BETWEEN_OXYGEN_REDRAW (2 sec)
- **Calls:** `draw_bar`, `MIN` macro
- **Notes:** Only redraws if `delay_time < 0` or forced. Clamps oxygen to PLAYER_MAXIMUM_SUIT_OXYGEN.

### update_weapon_panel
- **Signature:** `void HUD_Class::update_weapon_panel(bool force_redraw)`
- **Purpose:** Renders weapon selection display (single or dual shapes for multi-weapons) and weapon name text.
- **Inputs:** `force_redraw` ΓÇö skip dirty check
- **Outputs/Return:** None
- **Side effects:** Sets `weapon_is_dirty=false`, `ammo_is_dirty=true` (to update ammo display)
- **Calls:** `FillRect`, `DrawShapeAtXY`, `DrawText`, `get_player_desired_weapon`, `getcstr`, `get_item_name`, weapon_interface_definitions
- **Notes:** Handles multi-weapon logic (e.g., magnum + extra clip). Ball weapon is special-cased (checks which ball type player has).

### update_ammo_display
- **Signature:** `void HUD_Class::update_ammo_display(bool force_redraw)`
- **Purpose:** Updates both primary and secondary ammo displays.
- **Inputs:** `force_redraw`
- **Outputs/Return:** None
- **Side effects:** Sets `ammo_is_dirty=false`
- **Calls:** `draw_ammo_display_in_panel` (twice)

### update_inventory_panel
- **Signature:** `void HUD_Class::update_inventory_panel(bool force_redraw)`
- **Purpose:** Renders inventory screen (items, network stats, or game timer).
- **Inputs:** `force_redraw`
- **Outputs/Return:** None
- **Side effects:** Sets `INVENTORY_DIRTY_BIT=false`; may set `ForceUpdate=true`
- **Calls:** `calculate_player_item_array`, `get_header_name`, `draw_inventory_header`, `FillRect`, `calculate_player_rankings`, `draw_inventory_item`, and many game query functions
- **Notes:** Branches on `item_type` (network stats vs. item inventory). Network stats include game timer (countdown at 999 min), kill limit display, and player rankings.

### draw_bar
- **Signature:** `void HUD_Class::draw_bar(screen_rectangle *rectangle, short width, shape_descriptor top_piece, shape_descriptor full_bar, shape_descriptor background_texture)`
- **Purpose:** Renders a horizontal progress bar (e.g., energy or oxygen).
- **Inputs:** `rectangle` (bounds), `width` (fill), `top_piece` (end cap), `full_bar` (fill texture), `background_texture` (empty area)
- **Outputs/Return:** None
- **Side effects:** Calls virtual `DrawShape`/`DrawShapeAtXY`
- **Calls:** `offset_rect`, `DrawShape`, `DrawShapeAtXY`
- **Notes:** Handles narrow bars by clipping end cap. Uses screen rect source/dest for proper texture tiling.

### draw_ammo_display_in_panel
- **Signature:** `void HUD_Class::draw_ammo_display_in_panel(short trigger_id)`
- **Purpose:** Renders ammo counter as either energy bar (beam weapons) or grid of bullet shapes (ballistic).
- **Inputs:** `trigger_id` (_primary_interface_ammo or _secondary_interface_ammo)
- **Outputs/Return:** None
- **Side effects:** Calls virtual `FillRect`, `DrawShape`, `DrawShapeAtXY`
- **Calls:** `get_player_desired_weapon`, `get_player_weapon_ammo_count`, `PIN`, `offset_rect`, `DrawShapeAtXY`, `DrawShape`, `FillRect`
- **Notes:** Branches on ammo type (_uses_energy vs. _uses_bullets). Handles left-to-right vs. right-to-left ammo depletion. Pins ammo counts to valid ranges.

### draw_inventory_item
- **Signature:** `void HUD_Class::draw_inventory_item(char *text, short count, short offset, bool erase_first, bool valid_in_this_environment)`
- **Purpose:** Renders one inventory line (item name and count).
- **Inputs:** `text` (item name), `count`, `offset` (row index), `erase_first`, `valid_in_this_environment` (affects text color)
- **Outputs/Return:** None
- **Side effects:** Calls `FillRect`, `DrawText`
- **Calls:** `calculate_inventory_rectangle_from_offset`, `DrawText`, `FillRect`
- **Notes:** Always erases count digits; optionally erases entire row. Grays out items invalid in current environment (e.g., weapons in vacuum).

### draw_player_name
- **Signature:** `void HUD_Class::draw_player_name(void)`
- **Purpose:** Renders current player's name in the player name rectangle.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls virtual `DrawText`
- **Calls:** `get_player_data`, `get_interface_rectangle`, `DrawText`
- **Notes:** Uses player's color + PLAYER_COLOR_BASE_INDEX.

## Control Flow Notes
- **Entry point:** `update_everything` is called once per render frame.
- **Dirty flag pattern:** Each HUD subsystem checks its dirty flag; if set or force-redraw is true, it updates and clears the flag.
- **Lua texture palette override:** If Lua is active and palette is non-empty, standard HUD is skipped entirely; palette is displayed as a grid instead.
- **Network multiplayer mode:** When player_count > 1, network stats and player rankings replace inventory display.
- **Returns:** `ForceUpdate` signals to the caller whether the screen changed (used by rendering pipeline to optimize redraws).

## External Dependencies
- **Notable includes:** `HUDRenderer.h` (class definition), `lua_script.h` (conditional Lua texture palette), `config.h` (HAVE_LUA feature flag)
- **Implied externs:** `current_player`, `current_player_index`, `dynamic_world`, `interface_state`, `weapon_interface_definitions`, game query functions (`get_player_desired_weapon`, `calculate_player_item_array`, `get_player_weapon_ammo_count`, etc.)
- **Virtual methods (derived class):** `DrawShape`, `DrawShapeAtXY`, `DrawText`, `FillRect`, `FrameRect`, `DrawTexture`, `SetClipPlane`, `DisableClipPlane`, `update_motion_sensor`, `render_motion_sensor`, `draw_all_entity_blips`
