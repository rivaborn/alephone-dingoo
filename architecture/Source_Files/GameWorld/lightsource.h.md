# Source_Files/GameWorld/lightsource.h

## File Purpose
Defines the light source system for the Marathon game engine, including data structures for static light configuration and dynamic light state. Implements a state-machine-based lighting model with multiple transition functions and provides backwards compatibility with Marathon I light format.

## Core Responsibilities
- Define light data structures: static configuration (`static_light_data`), dynamic state (`light_data`), and legacy format (`old_light_data`)
- Specify lighting function types (constant, linear, smooth, flicker) for intensity transitions
- Manage a global light list and provide creation/query/control operations
- Support serialization and deserialization of light data (for save/load and network)
- Convert legacy Marathon I light format to Marathon II format for map compatibility

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `lighting_function_specification` | struct | Describes how light intensity transitions (function type, period, delta values, initial/final intensities) |
| `static_light_data` | struct | Immutable light configuration: type, flags, phase, six transition specifications (primary/secondary active/inactive, becoming active/inactive), tag |
| `light_data` | struct | Runtime light state: current intensity, phase, period, initial/final intensity, current state, wrapped `static_light_data` |
| `old_light_data` | struct | Marathon I light format (compatibility): flags, type, mode, phase, min/max intensity, period, current intensity |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `LightList` | `vector<light_data>` | global | All active lights in the map; size varies per map |

## Key Functions / Methods

### new_light
- Signature: `short new_light(struct static_light_data *data)`
- Purpose: Create a new light with given static configuration
- Inputs: Pointer to static light configuration
- Outputs/Return: Light index (short)
- Side effects: Appends to `LightList`
- Calls: (not visible in header)

### update_lights
- Signature: `void update_lights(void)`
- Purpose: Update all lights each frame; advance state machines and recalculate intensities
- Inputs: None (reads `LightList`)
- Outputs/Return: None
- Side effects: Modifies intensity and state of all lights in `LightList`
- Calls: (not visible in header)

### get_light_status / set_light_status
- Signature: `bool get_light_status(size_t light_index)` / `bool set_light_status(size_t light_index, bool active)`
- Purpose: Query or change active/inactive state of a single light
- Inputs: Light index, active boolean (for set)
- Outputs/Return: Boolean status
- Side effects: Modifies light state and may reset phase/period
- Calls: (not visible in header)

### set_tagged_light_statuses
- Signature: `bool set_tagged_light_statuses(short tag, bool new_status)`
- Purpose: Activate/deactivate all lights with a given tag (batch control)
- Inputs: Tag identifier, new status
- Outputs/Return: Boolean
- Side effects: Modifies state of multiple lights
- Calls: (not visible in header)

### get_light_intensity
- Signature: `_fixed get_light_intensity(size_t light_index)`
- Purpose: Retrieve current interpolated intensity of a light
- Inputs: Light index
- Outputs/Return: Fixed-point intensity value
- Calls: (not visible in header)

### get_light_data
- Signature: `light_data *get_light_data(const size_t light_index)`
- Purpose: Direct accessor for light state
- Inputs: Light index
- Outputs/Return: Pointer to `light_data`
- Calls: (not visible in header)

### Pack/Unpack functions
- Signatures: `uint8 *unpack_*_data(uint8 *Stream, T* Objects, size_t Count)` / `uint8 *pack_*_data(...)`
- Purpose: Serialize/deserialize light data (for save files, network, map data)
- Variants: `old_light_data`, `static_light_data`, `light_data`
- Inputs: Byte stream, array of objects, count
- Outputs/Return: Updated stream pointer
- Side effects: Reads/writes bytes

### convert_old_light_data_to_new
- Signature: `void convert_old_light_data_to_new(static_light_data* NewLights, old_light_data* OldLights, int Count)`
- Purpose: Migrate Marathon I light definitions to Marathon II format
- Inputs: Target and source light arrays, count
- Outputs/Return: None (modifies NewLights in-place)
- Side effects: Transforms old format to new
- Calls: (not visible in header)

**Notes on helpers:** `get_defaults_for_light_type()` returns a pre-configured static light for a given type.

## Control Flow Notes
Lights operate in a frame-based update loop. Each light transitions through states (becoming active ΓåÆ primary active ΓåÆ becoming inactive ΓåÆ primary inactive, with secondary variants) controlled by lighting function specifications. The `update_lights()` function is called each frame to advance timing and recompute intensities. States can be queried or set externally; lights with tags can be controlled in batches.

## External Dependencies
- `#include <vector>` ΓÇö C++ standard library for dynamic light collection
- Custom macros: `TEST_FLAG16()`, `SET_FLAG16()` (flag manipulation, defined elsewhere)
- Custom type: `_fixed` (likely fixed-point arithmetic type, defined elsewhere)
- Serialization functions suggest a structured binary format shared with map/save systems
