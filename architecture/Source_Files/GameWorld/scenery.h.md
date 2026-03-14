# Source_Files/GameWorld/scenery.h

## File Purpose
Header file declaring the scenery subsystem interface for the Aleph One game engine. Provides functions for creating, animating, and managing decorative/interactive world objects (scenery) such as platforms, lights, and fixtures. Supports XML-based configuration parsing for scenery properties.

## Core Responsibilities
- Initialize and manage the global scenery system
- Create new scenery objects with specified types and world locations
- Animate all active scenery each frame (flickering lights, moving platforms, etc.)
- Randomize scenery appearance/shape (both individual and all at once)
- Query scenery metadata (visual dimensions, graphical collections)
- Apply damage effects to scenery objects
- Provide XML parser interface for loading scenery definitions from configuration files

## Key Types / Data Structures
None defined in this file.

| Referenced Type | Kind | Purpose |
|---|---|---|
| `object_location` | struct | Position/orientation in world space (defined elsewhere) |
| `world_distance` | typedef | Distance unit in world coordinates (defined elsewhere) |
| `XML_ElementParser` | class | Base class for XML element parsing (from XML_ElementParser.h) |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_scenery
- Signature: `void initialize_scenery(void)`
- Purpose: Initialize the scenery subsystem at engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes internal scenery data structures
- Calls: (not visible from header)
- Notes: Called once during engine initialization

### new_scenery
- Signature: `short new_scenery(struct object_location *location, short scenery_type)`
- Purpose: Create and register a new scenery object in the game world
- Inputs: `location` (world position), `scenery_type` (type index)
- Outputs/Return: Object index (short) of newly created scenery
- Side effects: Allocates scenery object, updates global scenery state
- Calls: (not visible from header)
- Notes: Returns identifier for later reference

### animate_scenery
- Signature: `void animate_scenery(void)`
- Purpose: Update animation state of all active scenery objects
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies internal animation frame counters, updates object state
- Calls: (not visible from header)
- Notes: Called once per game frame

### randomize_scenery_shape / randomize_scenery_shapes
- Signature: `void randomize_scenery_shape(short object_index)` / `void randomize_scenery_shapes(void)`
- Purpose: Vary scenery appearance (e.g., randomly selected texture/shape variant)
- Inputs: `object_index` (for single) or none (for all)
- Outputs/Return: None
- Side effects: Modifies scenery visual state
- Calls: (not visible from header)
- Notes: Likely used for visual variety or Lua scripting (per comment)

### get_scenery_dimensions
- Signature: `void get_scenery_dimensions(short scenery_type, world_distance *radius, world_distance *height)`
- Purpose: Retrieve bounding dimensions for a scenery type
- Inputs: `scenery_type`, pointers to output variables
- Outputs/Return: None; `radius` and `height` filled by reference
- Side effects: None
- Calls: (not visible from header)
- Notes: Used for collision detection and rendering

### damage_scenery
- Signature: `void damage_scenery(short object_index)`
- Purpose: Apply damage effect to a scenery object
- Inputs: `object_index`
- Outputs/Return: None
- Side effects: Modifies scenery state (may switch to damaged collection)
- Calls: (not visible from header)

### get_scenery_collection / get_damaged_scenery_collection
- Signature: `bool get_scenery_collection(short scenery_type, short &collection)` / (damaged variant)
- Purpose: Retrieve graphical collection (sprite/shape resource) for a scenery type
- Inputs: `scenery_type`
- Outputs/Return: `bool` (success), `collection` filled by reference
- Side effects: None
- Calls: (not visible from header)
- Notes: Used by renderer to load correct graphics; damaged variant returns alternate collection

### Scenery_GetParser
- Signature: `XML_ElementParser *Scenery_GetParser()`
- Purpose: Obtain XML parser object for scenery configuration
- Inputs: None
- Outputs/Return: Pointer to `XML_ElementParser` instance
- Side effects: None
- Calls: (not visible from header)
- Notes: Enables dynamic scenery configuration loading from XML files

## Control Flow Notes
Fits into typical game loop as follows:
- **Initialization phase**: `initialize_scenery()` called once at engine startup
- **Level load phase**: `new_scenery()` called for each scenery object in level; `Scenery_GetParser()` used if loading from XML config
- **Frame/Update phase**: `animate_scenery()` called every frame; `damage_scenery()` called on collision/damage events
- **Utility**: Randomization and query functions called on demand (gameplay events, Lua scripting)

## External Dependencies
- `#include "XML_ElementParser.h"` ΓÇô XML parsing framework
- `object_location` ΓÇô defined elsewhere (game world location struct)
- `world_distance` ΓÇô defined elsewhere (coordinate/distance type)
- Comment notes Lua integration support (`ghs: allow Lua to add and delete scenery`)
