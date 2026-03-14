# Source_Files/RenderOther/motion_sensor.h

## File Purpose
Interface for the motion sensor HUD display in the Aleph One game engine. Defines display type enums, initialization/management functions, and XML configuration parsing for the in-game motion sensor that tracks nearby entities.

## Core Responsibilities
- Define entity type categories (friendly, alien, enemy) for motion sensor display
- Initialize motion sensor graphics with shape descriptors for each entity type
- Manage motion sensor state (reset, change detection, range adjustment)
- Provide XML parser integration for loading motion sensor configuration
- Track state changes to optimize rendering updates

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| MType_Friend, MType_Alien, MType_Enemy | enum constants | Classify entity types displayed on motion sensor |
| NUMBER_OF_MDISPTYPES | enum constant | Total count of display type categories |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_motion_sensor
- Signature: `void initialize_motion_sensor(shape_descriptor mount, shape_descriptor virgin_mounts, shape_descriptor alien, shape_descriptor _friend, shape_descriptor enemy, shape_descriptor network_compass, short side_length)`
- Purpose: Initialize motion sensor HUD with graphics assets for all entity types
- Inputs: shape descriptors for mount display, alien/friendly/enemy indicators, network compass, and sensor panel side length
- Outputs/Return: None (void)
- Side effects: Sets up internal motion sensor rendering state
- Calls: Defined elsewhere in motion_sensor.c
- Notes: Takes separate shape descriptors for each entity type; "_friend" naming avoids C++ keyword collision

### reset_motion_sensor
- Signature: `void reset_motion_sensor(short monster_index)`
- Purpose: Clear/reset motion sensor state for a specific entity
- Inputs: monster_index (entity identifier)
- Outputs/Return: None (void)
- Side effects: Clears internal state for that entity
- Calls: Defined elsewhere
- Notes: Likely called when entities are destroyed or despawned

### motion_sensor_has_changed
- Signature: `bool motion_sensor_has_changed(void)`
- Purpose: Query whether motion sensor display state has changed since last check
- Inputs: None
- Outputs/Return: true if state changed, false otherwise
- Side effects: None (query only)
- Calls: Defined elsewhere

### adjust_motion_sensor_range
- Signature: `void adjust_motion_sensor_range(void)`
- Purpose: Update motion sensor display range (zoom/scale)
- Inputs: None
- Outputs/Return: None (void)
- Side effects: Modifies internal sensor range state
- Calls: Defined elsewhere

### MotionSensor_GetParser
- Signature: `XML_ElementParser *MotionSensor_GetParser()`
- Purpose: Retrieve XML parser object for motion sensor configuration elements
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser for "motion_sensor" element
- Side effects: None
- Calls: Defined elsewhere
- Notes: Enables data-driven configuration via XML

## Control Flow Notes
Part of the rendering/HUD subsystem. Likely called during:
- **Init phase**: `initialize_motion_sensor()` at engine startup
- **Frame/update phase**: `motion_sensor_has_changed()`, `reset_motion_sensor()`, `adjust_motion_sensor_range()` during gameplay
- **Config phase**: `MotionSensor_GetParser()` during level/game data parsing

Tracks dynamic entity positions and renders them on the motion sensor display.

## External Dependencies
- `#include "XML_ElementParser.h"` ΓÇô XML configuration parsing infrastructure
- `shape_descriptor` type (defined elsewhere) ΓÇô graphics resource references
- Entity/monster system (undefined here) ΓÇô implicit dependency via monster_index parameter
