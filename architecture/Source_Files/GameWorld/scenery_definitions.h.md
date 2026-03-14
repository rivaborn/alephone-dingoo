# Source_Files/GameWorld/scenery_definitions.h

## File Purpose
Defines static data for environmental scenery objects in the game world. Provides flag constants, a scenery struct template, and a compiled array of 61 pre-configured scenery instances grouped by environment theme (lava, water, sewage, alien, Jjaro).

## Core Responsibilities
- Define scenery property flags (_scenery_is_solid, _scenery_is_animated, _scenery_can_be_destroyed)
- Provide the `scenery_definition` struct for static scenery configuration
- Supply a static array of 61 environment-specific scenery definitions
- Encode collision properties (radius, height) for each scenery object
- Link destroyed scenery to visual effects and replacement shapes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| scenery_definition | struct | Container for scenery properties: flags, renderable shape, collision geometry, destruction behavior |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| scenery_definitions | array [61] of `scenery_definition` | global | Lookup table of all pre-defined scenery objects; indexed during world initialization and rendering |
| NUMBER_OF_SCENERY_DEFINITIONS | macro constant | global | Hardcoded count of scenery entries (61) |

## Key Functions / Methods
None. This is a data-definition header (no executable functions).

## Control Flow Notes
Not inferable from this file. The `scenery_definitions` array is likely indexed by scenery type IDs during world loading and frame updates to retrieve collision and rendering properties.

## External Dependencies
- **Macros**: `BUILD_DESCRIPTOR(collection, index)` ΓÇö constructs shape references; defined elsewhere
- **Collections**: `_collection_scenery1` through `_collection_scenery5` ΓÇö shape/sprite collections referenced by indices
- **Effect constants**: `_effect_lava_lamp_breaking`, `_effect_water_lamp_breaking`, `_effect_sewage_lamp_breaking`, `_effect_alien_lamp_breaking`, `_effect_grenade_explosion`, `NONE` ΓÇö define what happens when scenery is destroyed
- **Scale constants**: `WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_ONE_FOURTH` ΓÇö unit measurements for collision radius and height
