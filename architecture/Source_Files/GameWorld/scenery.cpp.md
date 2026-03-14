# Source_Files/GameWorld/scenery.cpp

## File Purpose
Manages scenery objects (static decorative and destroyable world elements). Provides creation, animation, destruction, and XML-based configuration of scenery. Tracks which scenery needs per-frame animation updates and handles collision/rendering properties.

## Core Responsibilities
- Create scenery objects at map locations with proper ownership and collision flags
- Track and animate scenery requiring frame-by-frame updates
- Randomize scenery animation sequences on map load or per-object basis
- Apply damage and destruction effects to destroyable scenery
- Query scenery properties (dimensions, texture collections)
- Parse and apply XML-based scenery definition modifications

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `scenery_definition` | struct | Scenery properties: shape, destroyed_shape, flags, radius, height, destroyed_effect (defined in scenery_definitions.h) |
| `XML_SceneryObjectParser` | class | Parses scenery object XML elements; handles index, flags, radius, height, destruction attributes |
| `XML_SceneryShapesParser` | class | Parses shape descriptor elements (normal and destroyed variants) |
| `AnimatedSceneryObjects` | vector\<short\> | Tracks indices of scenery objects requiring animation updates each frame |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| AnimatedSceneryObjects | vector\<short\> | static | List of scenery object indices needing per-frame animation |
| original_scenery_definitions | scenery_definition* | static | Backup of default scenery definitions for XML reload capability |
| SceneryNormalParser | XML_SceneryShapesParser | static | Parser for normal (undestroyed) shape elements |
| SceneryDestroyedParser | XML_SceneryShapesParser | static | Parser for destroyed shape elements |
| SceneryObjectParser | XML_SceneryObjectParser | static | Parser for individual scenery object definitions |
| SceneryParser | XML_ElementParser | static | Root parser element for scenery XML tree |

## Key Functions / Methods

### initialize_scenery
- Signature: `void initialize_scenery(void)`
- Purpose: Pre-allocate capacity for animated scenery tracking
- Calls: `vector::reserve(32)`
- Notes: Called at engine startup

### new_scenery
- Signature: `short new_scenery(struct object_location *location, short scenery_type)`
- Purpose: Create scenery object at world location
- Inputs: world position/polygon, scenery type index
- Outputs/Return: object_index on success, NONE if definition invalid or object creation fails
- Side effects: Creates map object, sets ownership, solidity, permutation fields
- Calls: `get_scenery_definition()`, `new_map_object()`, `get_object_data()`, macros SET_OBJECT_OWNER/SOLIDITY

### animate_scenery
- Signature: `void animate_scenery(void)`
- Purpose: Advance animation frame for all tracked scenery
- Calls: `animate_object()` for each index in AnimatedSceneryObjects
- Notes: Called once per game tick

### randomize_scenery_shapes
- Signature: `void randomize_scenery_shapes(void)`
- Purpose: Initialize animation sequences for all map scenery
- Side effects: Clears, then repopulates AnimatedSceneryObjects by iterating all objects
- Calls: `randomize_object_sequence()`, GET_OBJECT_OWNER macro
- Notes: Called at map load time; adds to AnimatedSceneryObjects if sequence is random/animated

### damage_scenery
- Signature: `void damage_scenery(short object_index)`
- Purpose: Apply destruction effect to scenery
- Side effects: Swaps shape to destroyed variant; creates destruction effect; changes owner to _object_is_normal
- Calls: `new_effect()`, SET_OBJECT_OWNER macro
- Notes: Only if _scenery_can_be_destroyed flag set; skips effect if destroyed_effect == NONE

### get_scenery_dimensions
- Signature: `void get_scenery_dimensions(short scenery_type, world_distance *radius, world_distance *height)`
- Purpose: Query collision/rendering dimensions for a scenery type
- Outputs/Return: Sets radius/height via pointers (0,0 if definition not found)
- Calls: `get_scenery_definition()`

### get_scenery_collection, get_damaged_scenery_collection
- Signature: `bool get_scenery_collection(short scenery_type, short& collection)` (and damaged variant)
- Purpose: Retrieve texture collection index for scenery
- Outputs/Return: true + collection on success; false if undefined or not destroyable
- Calls: `GET_DESCRIPTOR_COLLECTION()` macro

### XML parser methods (Start, HandleAttribute, AttributesDone, ResetValues)
- Purpose: Parse/apply XML scenery configurations; back up and restore defaults
- Side effects: Modifies global scenery_definitions array; lazy-allocates original_scenery_definitions backup
- Calls: `ReadBoundedInt16Value()`, `ReadUInt16Value()`, `ReadInt16Value()`, `malloc()`, `free()`

### Scenery_GetParser
- Signature: `XML_ElementParser *Scenery_GetParser()`
- Purpose: Build and return root XML parser tree
- Outputs/Return: Pointer to configured SceneryParser
- Side effects: Attaches child parsers (SceneryObjectParser, shape parsers)
- Notes: Called during engine init to enable XML configuration support

## Control Flow Notes
**Startup:** `initialize_scenery()` reserves vector capacity. `Scenery_GetParser()` builds XML parser tree.

**Map load:** `randomize_scenery_shapes()` randomizes all scenery animation sequences; populates AnimatedSceneryObjects.

**Per-frame:** `animate_scenery()` called each tick to advance animation frames for tracked objects.

**Object placement:** `new_scenery()` creates objects; `randomize_scenery_shape()` randomizes individual animation.

**Destruction:** `damage_scenery()` applies destruction effect and changes object state.

**Configuration:** XML parser applies attribute overrides to scenery_definitions at load time.

## External Dependencies
- `cseries.h` ΓÇö core types, macros, utilities
- `<vector>` ΓÇö STL vector for dynamic scenery tracking
- `ShapesParser.h` ΓÇö shape descriptor parsing
- `map.h` ΓÇö object_data, new_map_object(), object owner/solidity macros, MAXIMUM_OBJECTS_PER_MAP
- `effects.h` ΓÇö new_effect()
- `scenery.h`, `scenery_definitions.h` ΓÇö scenery definitions array and types
- XML parser infrastructure (XML_ElementParser base class, Shape_GetParser(), Shape_SetPointer())
- Undefined in this file: `animate_object()`, `randomize_object_sequence()`, `GetMemberWithBounds()`, `new_map_object()` (defined elsewhere)
