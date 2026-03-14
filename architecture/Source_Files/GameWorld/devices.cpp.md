# Source_Files/GameWorld/devices.cpp

## File Purpose
Manages interactive control panels and switches in the game worldΓÇöwall-mounted devices that recharge player energy/oxygen, toggle lights and platforms, trigger computer terminals, and save games. Handles device state initialization, per-tick updates, player interaction detection, and visual/audio feedback.

## Core Responsibilities
- Initialize control panel states at level start based on linked lights/platforms
- Update recharging panels during game loop (shield/oxygen regeneration)
- Detect and process player action-key targets (platforms, control panels)
- Execute control panel state changes (toggle/activate/deactivate)
- Manage visual textures and sound effects for panel interactions
- Parse and apply XML configuration overrides for panel definitions and settings
- Support network game saves via pattern-buffer panels with double-click detection
- Validate panel toggle conditions (lighting requirements, item requirements, destruction flags)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `control_panel_definition` | struct | Static definition of a panel type: class, shapes, sounds, item requirement |
| `control_panel_settings_definition` | struct | Global recharge settings: reach distance, energy rates, horizontal weighting |
| `XML_CPSoundParser` | class | XML parser for panel sound indices within panel definitions |
| `XML_ControlPanelParser` | class | XML parser for individual panel definition overrides |
| `XML_ControlPanelsParser` | class | XML parser for global control-panel settings |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `control_panel_definitions[]` | `control_panel_definition` | static | Array of 45 predefined panel types (5 environment sets ├ù 9 types each) |
| `control_panel_settings` | `control_panel_settings_definition` | static | Global reach distance, energy caps, and recharge rates |
| `original_control_panel_definitions` | pointer | static | Backup of default definitions for XML reset |
| `original_control_panel_settings` | pointer | static | Backup of default settings for XML reset |
| `CPSoundParser`, `ControlPanelParser`, `ControlPanelsParser` | XML parsers | static | Singleton XML element parsers |

## Key Functions / Methods

### initialize_control_panels_for_level()
- Signature: `void initialize_control_panels_for_level(void)`
- Purpose: Set up initial states of all switches based on the lights and platforms they control
- Inputs: None (reads from global `map_sides` and `dynamic_world`)
- Outputs/Return: None; modifies side data in-place
- Side effects: Updates control panel status and textures for all sides; marks sides that reference invalid panel types as non-control-panels
- Calls: `GET_SIDE_CONTROL_PANEL()`, `get_control_panel_definition()`, `get_light_status()`, `platform_is_on()`, `SET_CONTROL_PANEL_STATUS()`, `set_control_panel_texture()`
- Notes: Handles three switch types specially (tag switch, light switch, platform switch); others use default state. Idiot-proofs against invalid panel type indices.

### update_control_panels()
- Signature: `void update_control_panels(void)`
- Purpose: Per-tick recharge logic for players actively using oxygen/shield recharge panels
- Inputs: None (reads global `players`, `dynamic_world`)
- Outputs/Return: None; modifies player suit_oxygen/suit_energy
- Side effects: Increments player oxygen/energy; marks display as dirty; plays sound; updates panel texture or stops activation sound
- Calls: `get_side_data()`, `get_control_panel_definition()`, `somebody_save_full_auto()`, `set_control_panel_texture()`, `play_control_panel_sound()`, `SoundManager::StopSound()`, `change_panel_state()`
- Notes: Full-auto save panels handled separately from recharge panels. Recharge continues only while player position and facing unchanged. Enforces maximum energy caps per panel type.

### update_action_key()
- Signature: `void update_action_key(short player_index, bool triggered)`
- Purpose: Resolve what the player is aiming at (if any) when action key is pressed
- Inputs: `player_index`, `triggered` (true if action was pressed)
- Outputs/Return: None; may trigger `player_touch_platform_state()` or `change_panel_state()` as side effect
- Side effects: Calls target-specific handlers (platform or panel); calls `vhalt()` on invalid target type
- Calls: `find_action_key_target()`, `player_touch_platform_state()`, `change_panel_state()`, `vhalt()`
- Notes: Only processes if `triggered` is true; ignored otherwise.

### change_panel_state()
- Signature: `void change_panel_state(short player_index, short panel_side_index)`
- Purpose: Execute the actual panel state change based on panel type and current state
- Inputs: `player_index`, `panel_side_index`
- Outputs/Return: None; modifies panel status, textures, global state (lights, platforms, player items)
- Side effects: Toggles lights/platforms, initiates computer terminal, saves game, plays sounds, updates panel texture, calls Lua hooks
- Calls: `get_side_data()`, `get_player_data()`, `get_control_panel_definition()`, `get_recharge_status()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `set_light_status()`, `try_and_change_platform_state()`, `enter_computer_interface()`, `try_and_subtract_player_item()`, `save_game_full_auto()`, `save_game()`, `play_control_panel_sound()`, Lua hooks
- Notes: Handles 9 panel classes with different logic paths. Network games use double-click detection for pattern-buffer (save) panels.

### set_control_panel_texture()
- Signature: `void set_control_panel_texture(struct side_data *side)`
- Purpose: Update the visual texture of a panel to match its current active/inactive state
- Inputs: `side` (control panel side)
- Outputs/Return: None; modifies `side->primary_texture.texture`
- Side effects: Changes displayed shape based on panel definition and status
- Calls: `get_control_panel_definition()`, `GET_CONTROL_PANEL_STATUS()`, `BUILD_DESCRIPTOR()`
- Notes: Texture selection depends on `active_shape` vs. `inactive_shape` in definition.

### try_and_toggle_control_panel()
- Signature: `void try_and_toggle_control_panel(short polygon_index, short line_index)`
- Purpose: Attempt to toggle a control panel when a monster or platform crosses into it (monster/environment-driven activation)
- Inputs: `polygon_index`, `line_index` (line being crossed)
- Outputs/Return: None; may toggle lights/platforms as side effect
- Side effects: Toggles tag switches, light switches, platform switches; plays sounds; calls Lua hooks
- Calls: `find_adjacent_side()`, `get_side_data()`, `switch_can_be_toggled()`, `get_control_panel_definition()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `set_light_status()`, `try_and_change_platform_state()`, `play_control_panel_sound()`, Lua hooks
- Notes: Used by platform/lighting system; distinct from player-initiated panel use.

### switch_can_be_toggled()
- Signature: `static bool switch_can_be_toggled(short side_index, bool player_hit)`
- Purpose: Validate whether a switch can be toggled (check lighting, item requirement, destruction flag)
- Inputs: `side_index`, `player_hit` (true if player triggered, false if monster/trigger)
- Outputs/Return: `true` if switch can be toggled; `false` otherwise
- Side effects: May destroy the switch (set `_side_is_control_panel` flag to false); plays "unusable" sound if player triggered an invalid switch
- Calls: `get_side_data()`, `get_control_panel_definition()`, `get_light_intensity()`, `play_control_panel_sound()`
- Notes: Player-hit switches require item in player inventory if `item != NONE`. Projectile-only switches reject player hits. Destroyable switches are removed on toggle.

### line_side_has_control_panel()
- Signature: `bool line_side_has_control_panel(short line_index, short polygon_index, short *side_index_with_panel)`
- Purpose: Find a control panel on a line's side facing the given polygon
- Inputs: `line_index`, `polygon_index`, `side_index_with_panel` (output)
- Outputs/Return: `true` if panel found; `side_index_with_panel` set to panel's side index
- Side effects: None
- Calls: `get_line_data()`, `get_side_data()`, `SIDE_IS_CONTROL_PANEL()`
- Notes: Checks both clockwise and counterclockwise sides of the line.

### find_action_key_target()
- Signature: `short find_action_key_target(short player_index, world_distance range, short *target_type)`
- Purpose: Ray-cast from player forward direction to find targetable objects (platforms, control panels) within range
- Inputs: `player_index`, `range`, `target_type` (output: `_target_is_platform` or `_target_is_control_panel`)
- Outputs/Return: Object index (platform or side index); `NONE` if nothing found
- Side effects: None
- Calls: `get_player_data()`, `ray_to_line_segment()`, `find_line_crossed_leaving_polygon()`, `get_line_data()`, `find_adjacent_polygon()`, `get_polygon_data()`, `line_is_within_range()`, `platform_is_legal_player_target()`, `line_side_has_control_panel()`, `switch_can_be_toggled()`
- Notes: Traces ray through polygon boundaries until hitting a platform or panel within range, or leaving the map.

### get_control_panel_definition()
- Signature: `control_panel_definition *get_control_panel_definition(const short control_panel_type)`
- Purpose: Look up a control panel definition by type index with bounds checking
- Inputs: `control_panel_type`
- Outputs/Return: Pointer to definition; `NULL` if out of range
- Side effects: None
- Calls: `GetMemberWithBounds()`
- Notes: Idiot-proofs against invalid indices; callers should check for `NULL`.

### XML_ControlPanelParser::AttributesDone() / XML_ControlPanelsParser::HandleAttribute()
- Purpose: Parse XML attributes and apply overrides to panel definitions and settings
- Inputs: XML attributes (tag/value pairs)
- Outputs/Return: `true` on valid parse; `false` on error
- Side effects: Modifies `control_panel_definitions[]` or `control_panel_settings`
- Calls: `ReadBoundedInt16Value()`, `ReadFloatValue()`, etc.
- Notes: Supports per-panel overrides and global settings; backs up originals for reset.

## Control Flow Notes
**Initialization (entering level):**  
`initialize_control_panels_for_level()` runs once per level, scanning all sides to set up panel initial states based on linked lights and platforms.

**Game loop (per tick):**  
`update_control_panels()` recharges active players; `update_action_key()` handles player action presses.

**External activation (lights/platforms trigger panels):**  
`try_and_toggle_control_panel()` called by platform/light systems; `assume_correct_switch_position()` syncs panel state after linked element changes.

**Panel state propagation:**  
Control panels toggle linked tag switches, which cascade to update lights and platforms via their respective managers.

## External Dependencies
- **map.h**: `side_data`, `line_data`, `polygon_data`, `map_sides`, `dynamic_world`, map accessor/utility functions
- **monsters.h**: `get_monster_data()`, `get_monster_dimensions()`
- **player.h**: `player_data`, `players`, `local_player`, `get_player_data()`, `try_and_subtract_player_item()`
- **platforms.h**: `try_and_change_platform_state()`, `try_and_change_tagged_platform_states()`, `platform_is_on()`, `get_polygon_data()`
- **SoundManager.h**: `SoundManager::instance()`, `PlaySound()`, `StopSound()`
- **computer_interface.h**: `enter_computer_interface()`, `calculate_level_completion_state()`
- **lightsource.h**: `set_light_status()`, `set_tagged_light_statuses()`, `get_light_status()`, `get_light_intensity()`
- **items.h**: Item management (via `try_and_subtract_player_item()`)
- **interface.h**: `get_game_state()`, `save_game()`, `save_game_full_auto()`, `screen_printf()`
- **XML_ElementParser.h**: XML parsing base classes

---

**Notes:**  
- This file integrates tightly with platforms, lights, saves, and terminalsΓÇöchanges to panel logic may cascade.
- XML override system allows mods to customize panel behavior without code changes.
- Lua scripting hooks are conditional and mostly commented, suggesting legacy/optional integration.
- Network saves via pattern buffer use double-click detection to distinguish intentional overwrites.
