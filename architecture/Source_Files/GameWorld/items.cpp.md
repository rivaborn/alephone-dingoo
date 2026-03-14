# Source_Files/GameWorld/items.cpp

## File Purpose
Manages item placement, collection, and consumption in the game world. Handles item spawning, player pickup mechanics, item animation, and XML configuration of item properties for the Aleph One Marathon engine.

## Core Responsibilities
- Item creation and map placement via `new_item()`
- Player item acquisition through proximity-based pickup (`swipe_nearby_items()`) and manual collection (`get_item()`)
- Item effect application and inventory management (`try_and_add_player_item()`)
- Item animation and shape randomization (`animate_items()`)
- Zone-based item triggering for activation (`trigger_nearby_items()`)
- Inventory queries and state tracking (`calculate_player_item_array()`, `count_inventory_lines()`, `find_player_ball_color()`)
- Item validation against game environment and network mode
- XML-driven item definition configuration and modification

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `item_definition` | struct | Defines item properties: name IDs, max count, valid environments, kind, base shape (defined in item_definitions.h) |
| `object_data` | struct | In-world object representation; items use owner flag `_object_is_item` with type in permutation |
| `player_data` | struct | Contains `items[]` array tracking inventory counts per item type |
| `XML_ItemParser` | class | Parses and applies XML item element modifications to global definitions |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `item_definitions` | `item_definition[]` | global (external) | Item property lookup table indexed by type |
| `original_item_definitions` | `item_definition*` | static | Backup of item definitions before XML parsing for reset capability |
| `ItemParser` | `XML_ItemParser` | static | Parser instance for XML `<item>` elements |
| `ItemsParser` | `XML_ElementParser` | static | Root container parser for `<items>` XML element |

## Key Functions / Methods

### new_item
- **Signature**: `short new_item(struct object_location *location, short type)`
- **Purpose**: Spawn a new item object in the game world at specified location
- **Inputs**: location (3D position + polygon), item type ID
- **Outputs/Return**: Object index (NONE if creation failed)
- **Side effects**: Creates global map object; sets owner flag and permutation; calls Lua item creation hook; may set invisibility flag for network-only items in single-player
- **Calls**: `get_item_definition()`, `new_map_object()`, `get_object_data()`, `SET_OBJECT_OWNER()`, `object_was_just_added()`, `L_Call_Item_Created()`
- **Notes**: Filters items by game mode (network vs single-player); balls only added if current game supports them; network-only items auto-hidden in single-player

### swipe_nearby_items
- **Signature**: `void swipe_nearby_items(short player_index)`
- **Purpose**: Automatically collect all items within arm's reach and line-of-sight
- **Inputs**: player index
- **Outputs/Return**: None
- **Side effects**: Modifies player inventory; removes item objects from map; restarts search on each collection
- **Calls**: `get_polygon_data()`, `get_map_indexes()`, `get_object_data()`, `guess_distance2d()`, `get_monster_dimensions()`, `test_item_retrieval()`, `get_item()`, `get_item_kind()`
- **Notes**: Checks current + neighboring polygons within MAXIMUM_ARM_REACH; verifies line-of-sight and height constraints; recursively retries after successful pickup

### try_and_add_player_item
- **Signature**: `bool try_and_add_player_item(short player_index, short type)`
- **Purpose**: Attempt to give item to player (apply effect or add to inventory)
- **Inputs**: player index, item type
- **Outputs/Return**: true if item was successfully acquired
- **Side effects**: Modifies player inventory array; applies powerup effects; plays audio/visual feedback; calls Lua hook; may reload weapons; marks interface dirty
- **Calls**: `get_item_definition()`, `get_player_data()`, `legal_player_powerup()`, `process_player_powerup()`, `find_player_ball_color()`, `process_new_item_for_reloading()`, `mark_player_inventory_as_dirty()`, `Sound_GotPowerup()`, `Sound_GotItem()`, `start_fade()`, `L_Call_Got_Item()`, `SoundManager::instance()->PlayLocalSound()`
- **Notes**: Different handling for powerups (consumed immediately), balls (max 1; returns to map if own team's ball outside base), weapons/ammo/items (respects max count except ammo on total carnage); skips knife

### animate_items
- **Signature**: `void animate_items(void)`
- **Purpose**: Update animation frames for all items in world each frame
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Updates object sequence/phase; randomizes non-animated item orientations once
- **Calls**: `get_object_data()`, `get_item_kind()`, `get_item_definition()`, `get_shape_animation_data()`, `randomize_object_sequence()`, `animate_object()`
- **Notes**: Uses object `facing` field as flag for randomization (set to NONE after first randomize); handles both animated and static shapes

### trigger_nearby_items
- **Signature**: `void trigger_nearby_items(short polygon_index)`
- **Purpose**: Activate (teleport in) all invisible items in connected zone
- **Inputs**: trigger polygon index
- **Outputs/Return**: None
- **Side effects**: Changes invisibility flag; teleports objects
- **Calls**: `flood_map()`, `item_trigger_cost_function()`, `get_polygon_data()`, `get_object_data()`, `teleport_object_in()`
- **Notes**: Uses flood fill to traverse polygons; stops at zone borders (cost -1); activates all invisible items found

### XML_ItemParser::AttributesDone
- **Signature**: `bool XML_ItemParser::AttributesDone()`
- **Purpose**: Validate parsed XML item attributes and apply to global definitions
- **Inputs**: None (uses member `Data` and `IsPresent[]` from prior `HandleAttribute()` calls)
- **Outputs/Return**: true if validation passed
- **Side effects**: Modifies global `item_definitions[]`; backs up originals on first call
- **Calls**: `Shape_SetPointer()`
- **Notes**: Index attribute mandatory; backs up definitions once for reset capability; selectively updates fields based on presence flags

**Other functions** (documented in notes):
- `get_item_definition()` / `get_item_definition_external()` ΓÇô definition lookup with bounds checking
- `find_player_ball_color()` ΓÇô scan inventory for ball items
- `get_item_name()` ΓÇô retrieve localized item name string
- `calculate_player_item_array()` ΓÇô filter inventory by type
- `count_inventory_lines()` ΓÇô UI display line count
- `test_item_retrieval()` ΓÇô line-of-sight check through polygons/platforms
- `item_trigger_cost_function()` ΓÇô flood-fill cost callback (stops at zone borders)

## Control Flow Notes

**Initialization** (`initialize_items`): Currently a stub (empty).

**Per-frame animation** (`animate_items`): Called during world update to advance animation frames; randomizes non-animated items on first frame of level.

**Player update** (implicit): `swipe_nearby_items()` called when player moves to auto-collect proximal items; `try_and_add_player_item()` called to apply item effects.

**Event triggers** (polygon entry): `trigger_nearby_items()` called when player enters item-trigger polygons to activate zone items.

**Configuration** (startup): XML parsing via `Items_GetParser()` allows MML (Marathon Markup Language) overrides of item definitions before level load.

## External Dependencies

- **Geometry/world**: `map.h` (polygons, objects, lines, platforms)
- **Entities**: `monsters.h`, `player.h` (entity data, inventory)
- **Systems**: `weapons.h` (reload), `SoundManager.h` (audio), `fades.h` (screen effects)
- **State**: `network_games.h` (multiplayer detection), `lua_script.h` (scripting callbacks)
- **Global**: `item_definitions` array, `dynamic_world`, `static_world`, `objects` array, game environment flags
