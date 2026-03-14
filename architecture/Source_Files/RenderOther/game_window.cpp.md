# Source_Files/RenderOther/game_window.cpp

## File Purpose
Manages the game window HUD (Heads-Up Display) for both software and OpenGL rendering modes. Handles HUD initialization, frame updates, weapon/ammo display state tracking, inventory navigation, and XML-driven configuration of interface element layouts (weapons, ammo counters, colors, fonts).

## Core Responsibilities
- Initialize HUD buffer for software rendering (SDL surface)
- Update and draw interface panels each frame, respecting dirty-flag optimization
- Track and propagate HUD element dirty states (weapon, ammo, shields, oxygen)
- Manage player inventory screen navigation and scrolling
- Parse and apply XML configuration for weapon/ammo display positioning and appearance
- Initialize motion sensor display with graphics resources
- Coordinate rendering between software (SDL) and OpenGL paths
- Manage microphone state for network audio

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `interface_state_data` | struct | Caches dirty flags for weapon, ammo, shield, oxygen HUD elements |
| `weapon_interface_data` | struct | Layout parameters for one weapon's HUD panel (position, ammo slots, multi-weapon variants) |
| `weapon_interface_ammo_data` | struct | Display metrics for ammo counter within a weapon panel |
| `HUD_SW_Class` | class | Software HUD renderer; inherits from HUD_Class, implements SDL-based drawing |
| `XML_AmmoDisplayParser` | class | Parses `<ammo>` MML tags to customize ammo display layout |
| `XML_WeaponDisplayParser` | class | Parses `<weapon>` MML tags to customize weapon panel layout |
| `XML_InterfaceParser` | class | Root parser for `<interface>` MML element; aggregates child parsers |
| `XML_VidmasterParser` | class | Parses `<vidmaster>` tags for secret/easter-egg string set indices |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MotionSensorActive` | bool | global | Whether motion sensor is enabled; set via XML parser |
| `interface_state` | interface_state_data | global | Dirty flags for each HUD element (weapon, ammo, shield, oxygen) |
| `weapon_interface_definitions[10]` | weapon_interface_data[] | global | Layout data for all 10 weapons; initialized at compile-time, overridable by XML |
| `original_weapon_interface_definitions` | weapon_interface_data* | static | Backup copy of original weapon layouts for XML reset functionality |
| `HUD_SW` | HUD_SW_Class | global | Singleton software HUD renderer |
| `HUD_Buffer` | SDL_Surface* | global (extern) | Double-buffer surface for HUD; allocated on first use |
| `static_hud_pict` (in `draw_panels`) | SDL_Surface* | static/local | Cached background HUD image (loaded once, reused each frame) |
| `hud_pict_not_found` (in `draw_panels`) | bool | static/local | Flag indicating if HUD background image failed to load from resources |
| `VidmasterParser`, `AmmoDisplayParser`, `WeaponDisplayParser`, `InterfaceParser` | XML_*Parser | static | Singleton parser instances for XML configuration |

## Key Functions / Methods

### initialize_game_window
- **Signature**: `void initialize_game_window(void)`
- **Purpose**: Initialize motion sensor display with resource descriptors at engine startup
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: Calls underlying motion sensor init with graphics collection references
- **Calls**: `initialize_motion_sensor()`
- **Notes**: Passes 6 BUILD_DESCRIPTOR references for sensor imagery (mount, alien, friend, enemy, compass)

### draw_interface
- **Signature**: `void draw_interface(void)`
- **Purpose**: Render the complete interface frame (non-full-screen mode only)
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: Renders panel graphics; early-exits if OpenGL mode active
- **Calls**: `alephone::Screen::instance()->openGL()`, `game_window_is_full_screen()`, `draw_panels()`, `validate_world_window()`
- **Notes**: Skipped entirely when OpenGL rendering is active

### update_interface
- **Signature**: `void update_interface(short time_elapsed)`
- **Purpose**: Update dirty HUD elements; perform full redraw if time_elapsed == NONE
- **Inputs**: time_elapsed ΓÇö elapsed milliseconds since last update, or NONE to force complete redraw
- **Outputs/Return**: void
- **Side effects**: Allocates HUD buffer if needed; updates software HUD renderer state; may request HUD redraw
- **Calls**: `alephone::Screen::instance()->openGL()`, `reset_motion_sensor()`, `game_window_is_full_screen()`, `ensure_HUD_buffer()`, `HUD_SW.update_everything()`, `RequestDrawingHUD()`, port-management functions
- **Notes**: Entirely skipped in OpenGL mode (except resetting motion sensor on full update). Uses dirty-flag optimization: only redraws HUD if `update_everything()` returns true or full redraw is requested.

### mark_weapon_display_as_dirty, mark_ammo_display_as_dirty, mark_shield_display_as_dirty, mark_oxygen_display_as_dirty
- **Signature**: `void mark_*_display_as_dirty(void)` (4 variants)
- **Purpose**: Flag specific HUD element for redraw on next frame
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: Sets one bit in `interface_state` (weapon_is_dirty, ammo_is_dirty, etc.)
- **Calls**: None
- **Notes**: Called immediately after game state changes (weapon switch, ammo consumed, shield damage). Trivial setters; actual redraw happens in `update_interface()`.

### mark_player_inventory_screen_as_dirty
- **Signature**: `void mark_player_inventory_screen_as_dirty(short player_index, short screen)`
- **Purpose**: Force-switch player's inventory view to a specific category and mark dirty
- **Inputs**: player_index (which player), screen (item type constant or `_network_statistics`)
- **Outputs/Return**: void
- **Side effects**: Updates player.interface_flags (packs screen into bits), resets interface_decay timer
- **Calls**: `get_player_data()`, `set_current_inventory_screen()`, `SET_INVENTORY_DIRTY_STATE()`
- **Notes**: Used to forcibly show the inventory screen corresponding to a picked-up item type

### mark_player_inventory_as_dirty
- **Signature**: `void mark_player_inventory_as_dirty(short player_index, short dirty_item)`
- **Purpose**: Mark inventory dirty; intelligently switch to item's category if it differs from current view
- **Inputs**: player_index, dirty_item (item descriptor, or NONE to skip category switch)
- **Outputs/Return**: void
- **Side effects**: May switch inventory screen; sets dirty flag
- **Calls**: `get_player_data()`, `get_item_kind()`, `GET_CURRENT_INVENTORY_SCREEN()`, `set_current_inventory_screen()`, `SET_INVENTORY_DIRTY_STATE()`
- **Notes**: Smart: auto-navigates to new inventory tab only if item is not a powerup and differs from current view. Does *not* switch to network-statistics screen.

### scroll_inventory
- **Signature**: `void scroll_inventory(short dy)`
- **Purpose**: Scroll inventory view up or down to next non-empty item category
- **Inputs**: dy ΓÇö direction (positive = up/next, negative = down/previous)
- **Outputs/Return**: void
- **Side effects**: Updates current inventory screen; marks dirty
- **Calls**: `GET_CURRENT_INVENTORY_SCREEN()`, `calculate_player_item_array()`, `SET_INVENTORY_DIRTY_STATE()`
- **Notes**: Wraps around modulo. Skips empty item categories. Network-statistics screen acts as an extra screen in multiplayer. Uses modulo math to cycle: `(current + index) % mod_value`.

### set_current_inventory_screen (static)
- **Signature**: `static void set_current_inventory_screen(short player_index, short screen)`
- **Purpose**: Internal helper to update player's active inventory category
- **Inputs**: player_index, screen (0ΓÇô6: item types or network_statistics)
- **Outputs/Return**: void
- **Side effects**: Packs screen into lower bits of player.interface_flags; sets interface_decay to 5 seconds
- **Calls**: `get_player_data()`
- **Notes**: Asserts screen is in range [0, 7). The decay timer auto-hides the inventory after 5 ticks.

### ensure_HUD_buffer
- **Signature**: `void ensure_HUD_buffer(void)`
- **Purpose**: Allocate SDL surface for HUD rendering if not already present
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: Creates `HUD_Buffer` global (640├ù480, 8-bit SDL_Surface); calls `alert_user()` with fatal error on allocation failure
- **Calls**: `SDL_CreateRGBSurface()`, `SDL_DisplayFormat()`, `SDL_FreeSurface()`, `alert_user()`
- **Notes**: Idempotent: safe to call multiple times. Uses SWSURFACE, then converts to display format. Will crash game on malloc failure.

### draw_panels
- **Signature**: `void draw_panels(void)`
- **Purpose**: Draw complete HUD (static background + dynamic elements) to HUD buffer for non-OpenGL mode
- **Inputs**: None
- **Outputs/Return**: void
- **Side effects**: Ensures HUD buffer allocated; loads and caches static HUD image; renders HUD_SW; requests redraw
- **Calls**: `alephone::Screen::instance()->openGL()`, `ensure_HUD_buffer()`, `get_picture_resource_from_images()`, `picture_to_surface()`, `SDL_BlitSurface()`, `HUD_SW.update_everything()`, `RequestDrawingHUD()`
- **Notes**: Static HUD background loaded once and cached in local static variable. Early-exits if OpenGL active. Includes Lua texture-palette hack for gp2x/dingoo platforms.

### mark_interface_collections
- **Signature**: `void mark_interface_collections(bool loading)`
- **Purpose**: Mark interface graphics collection for loading or unloading
- **Inputs**: loading ΓÇö true to load, false to unload
- **Outputs/Return**: void
- **Side effects**: Updates collection resource-manager state
- **Calls**: `mark_collection_for_loading()` or `mark_collection_for_unloading()`
- **Notes**: Simple pass-through wrapper for ternary resource management.

### mark_player_network_stats_as_dirty
- **Signature**: `void mark_player_network_stats_as_dirty(short player_index)`
- **Purpose**: Switch player to network-statistics inventory screen and mark dirty (if enabled)
- **Inputs**: player_index
- **Outputs/Return**: void
- **Side effects**: May switch inventory screen; marks dirty
- **Calls**: `GET_GAME_OPTIONS()`, `get_player_data()`, `set_current_inventory_screen()`, `SET_INVENTORY_DIRTY_STATE()`
- **Notes**: Only applies if game option `_live_network_stats` is set. No-op otherwise.

### set_interface_microphone_recording_state
- **Signature**: `void set_interface_microphone_recording_state(bool state)`
- **Purpose**: Enable or disable network microphone recording
- **Inputs**: state ΓÇö true to record, false to mute
- **Outputs/Return**: void
- **Side effects**: Updates network audio subsystem
- **Calls**: `set_network_microphone_state()` (if `DISABLE_NETWORKING` not defined)
- **Notes**: Entirely no-op if networking disabled at compile-time.

---

**XML Parser Methods** (summarized as trivial):
- `XML_VidmasterParser`, `XML_AmmoDisplayParser`, `XML_WeaponDisplayParser`, `XML_InterfaceParser`: Each implements `Start()`, `HandleAttribute()`, `AttributesDone()` to parse MML configuration tags and apply customizations to weapon/ammo/interface layouts and colors. `Interface_GetParser()` constructs the parser tree.

## Control Flow Notes
**Startup**: `initialize_game_window()` sets up motion sensor.

**Per-frame rendering loop**: `update_interface(time_elapsed)` checks for OpenGL mode, updates HUD buffer, and flags redraw. `draw_interface()` renders frame.

**On game events**: `mark_*_as_dirty()` functions flag which elements need redraw (called by game logic on state change).

**Inventory**: `scroll_inventory()` and `mark_player_inventory_as_dirty()` coordinate inventory navigation; `set_current_inventory_screen()` packs category into player flags.

**Configuration**: XML parsers called at level/config load to customize layouts and colors.

**Rendering**: Operates in **software SDL mode** (with HUD buffer) or **OpenGL mode** (early-exit paths throughout). Lua HUD hack included for embedded platforms.

## External Dependencies
- `cseries.h` ΓÇö core engine types, macros
- `HUDRenderer_SW.h` ΓÇö `HUD_SW_Class` (software HUD renderer)
- `game_window.h` ΓÇö public interface (declarations)
- `ColorParser.h`, `FontHandler.h` ΓÇö XML-based config
- `screen.h` ΓÇö `alephone::Screen` singleton, rendering mode queries
- `shell.h` ΓÇö shell utilities, screen_mode_data
- `preferences.h` ΓÇö game settings
- `images.h` ΓÇö `get_picture_resource_from_images()`, `picture_to_surface()`
- `network_sound.h` ΓÇö `set_network_microphone_state()`
- SDL headers (via cseries.h) ΓÇö `SDL_Surface`, `SDL_Rect`, `SDL_BlitSurface()`, etc.
- OpenGL headers (conditional) ΓÇö `gl.h` / `GL/gl.h`

**Defined elsewhere** (called but not defined here):
- `initialize_motion_sensor()`, `reset_motion_sensor()`
- `draw_panels()` (forward-declared, then re-defined later in same file)
- `validate_world_window()`
- `game_window_is_full_screen()`
- `get_player_data()`, `calculate_player_item_array()`, `get_item_kind()`
- `SET_INVENTORY_DIRTY_STATE()`, `GET_CURRENT_INVENTORY_SCREEN()`, `GET_GAME_OPTIONS()`
- `RequestDrawingHUD()`, `RequestDrawingTerm()`
- `mark_collection_for_loading/unloading()`
- `alert_user()`, `_set_port_to_HUD()`, `_restore_port()`
- Color/font parsing utilities
