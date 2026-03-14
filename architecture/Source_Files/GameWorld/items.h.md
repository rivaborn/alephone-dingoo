# Source_Files/GameWorld/items.h

## File Purpose
Header file declaring item system interfaces for the Aleph One game engine. Defines item type enumerations (weapons, ammunition, powerups, balls), item management functions, and XML configuration support for runtime item customization.

## Core Responsibilities
- Define item class types (weapon, ammunition, powerup, item, weapon_powerup, ball) and specific item IDs
- Declare item creation and placement functions
- Provide player inventory management (add, retrieve, count items)
- Implement item triggering and map-based item queries
- Support XML-based item configuration and initialization
- Handle item animation and frame updates

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (unnamed enum) | enum | Item type classes: `_weapon`, `_ammunition`, `_powerup`, etc. |
| (unnamed enum) | enum | Specific item IDs: `_i_knife`, `_i_magnum`, `_i_plasma_pistol`, etc.; includes ball variants |

*Referenced but not defined:*  
- `struct object_location` ΓÇö passed to `new_item()`  
- `struct item_definition` ΓÇö configuration data (defined elsewhere)  
- `XML_ElementParser` ΓÇö XML parsing class from XML_ElementParser.h

## Global / File-Static State
None (header only; state managed in items.c).

## Key Functions / Methods

### new_item
- **Signature:** `short new_item(struct object_location *location, short item_type)`
- **Purpose:** Create and place a new item in the game world at a specified location.
- **Inputs:** Location struct, item type ID
- **Outputs/Return:** Item ID (short), or error code
- **Side effects:** Allocates item object, updates world state
- **Calls:** Not visible from header

### try_and_add_player_item
- **Signature:** `bool try_and_add_player_item(short player_index, short type)`
- **Purpose:** Attempt to add an item to a player's inventory (respects carrying capacity).
- **Inputs:** Player index, item type
- **Outputs/Return:** Boolean success/failure
- **Side effects:** Modifies player inventory if successful
- **Calls:** Not visible from header

### calculate_player_item_array
- **Signature:** `void calculate_player_item_array(short player_index, short type, short *items, short *counts, short *array_count)`
- **Purpose:** Query player inventory for items of a given type/class.
- **Inputs:** Player index, item type filter; output buffers (items, counts, array_count)
- **Outputs/Return:** Populates `items` and `counts` arrays; sets `array_count`
- **Side effects:** None
- **Calls:** Not visible from header

### swipe_nearby_items
- **Signature:** `void swipe_nearby_items(short player_index)`
- **Purpose:** Automatically collect all items adjacent to or near the player.
- **Inputs:** Player index
- **Outputs/Return:** None
- **Side effects:** Transfers items to player inventory, removes from world
- **Calls:** Not visible from header

### trigger_nearby_items
- **Signature:** `void trigger_nearby_items(short polygon_index)`
- **Purpose:** Activate/trigger items in a given polygon (e.g., activate switches, pressable powerups).
- **Inputs:** Polygon index (spatial location)
- **Outputs/Return:** None
- **Side effects:** May modify item state or world state
- **Calls:** Not visible from header

### find_player_ball_color
- **Signature:** `short find_player_ball_color(short player_index)`
- **Purpose:** Retrieve the color of a ball item carried by the player (used in ball-based game modes).
- **Inputs:** Player index
- **Outputs/Return:** Ball color enum, or `NONE` if player has no ball
- **Side effects:** None
- **Calls:** Not visible from header

### get_item_definition_external
- **Signature:** `struct item_definition *get_item_definition_external(const short type)`
- **Purpose:** Retrieve configuration/metadata for a given item type (shape, physics, etc.).
- **Inputs:** Item type ID
- **Outputs/Return:** Pointer to item definition struct
- **Side effects:** None
- **Calls:** Not visible from header

### initialize_items
- **Signature:** `void initialize_items(void)`
- **Purpose:** Initialize the item system on game startup; load XML configurations.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Loads and configures all item definitions
- **Calls:** Not visible from header

### animate_items
- **Signature:** `void animate_items(void)`
- **Purpose:** Update item animations and state each frame.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates animation frames, may trigger state transitions
- **Calls:** Not visible from header

### Items_GetParser
- **Signature:** `XML_ElementParser *Items_GetParser()`
- **Purpose:** Obtain the XML parser object for parsing item configuration files.
- **Inputs:** None
- **Outputs/Return:** Pointer to XML parser for items element
- **Side effects:** None
- **Calls:** Not visible from header

## Control Flow Notes
- **Initialization:** `initialize_items()` called at game startup to load item definitions from XML.
- **Per-frame:** `animate_items()` called each frame to update item animations and visual state.
- **Item creation:** `new_item()` called when items spawn (pickups, drops, level placement).
- **Player interaction:** `swipe_nearby_items()`, `trigger_nearby_items()`, and `try_and_add_player_item()` respond to player actions.
- **Query:** `unretrieved_items_on_map()` used to check win conditions; `calculate_player_item_array()` used for UI and inventory checks.

## External Dependencies
- **XML_ElementParser.h** ΓÇö XML parsing infrastructure for item configuration
- **cstypes.h** (via XML_ElementParser.h) ΓÇö Custom type definitions
- **Defined elsewhere:**  
  - `struct object_location` ΓÇö World position/placement data  
  - `struct item_definition` ΓÇö Item configuration (shape, behavior, properties)  
  - Item enums and functions implemented in items.c
