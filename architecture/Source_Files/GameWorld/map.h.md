# Source_Files/GameWorld/map.h

## File Purpose
Central header for the game world and map system, defining map geometry, object placement, game state, and physics structures. Contains data structure definitions, enums for game mechanics, and function prototypes for world initialization, object management, spatial queries, and rendering support.

## Core Responsibilities
- **Map Geometry**: Define and manage polygons, sides, lines, and endpoints that form the 3D world
- **Game Objects**: Create, track, and manage objects (monsters, items, effects) within the world
- **Game State**: Store static map properties and dynamic runtime state (tick count, counts, player info)
- **Physics/Collision**: Provide geometric queries (point-in-polygon, line-of-sight, distance calculations)
- **Audio**: Manage ambient and random sound placements with directional/non-directional support
- **Damage System**: Define damage types, flags, and propagation mechanics
- **Game Configuration**: Define game modes, mission types, difficulty levels, entry points, and player spawns
- **Serialization**: Pack/unpack data structures for save/load and network transmission

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| **damage_definition** | struct | Damage properties: type, base/random amount, scaling |
| **map_object** | struct | Initial object placement on map (type, location, facing, flags) |
| **object_data** | struct | Runtime object state (location, animation, transfer mode, sound pitch) |
| **polygon_data** | struct | Floor/ceiling polygon with adjacency, textures, lights, media, sound sources; contains object list head |
| **line_data** | struct | Line segment between endpoints; links to adjacent polygons and their sides |
| **side_data** | struct | Textured surface on a line; handles control panels, textures, transfer modes, lighting |
| **endpoint_data** | struct | 3D vertex with elevation flags and supporting polygon info |
| **object_location** | struct | 3D point + polygon index + yaw/pitch angles; used for spawning |
| **player_start_data** | struct | Player spawn configuration (team, identifier with flags, color, name) |
| **game_data** | struct | Game parameters (type, options, cheats, kill limit, difficulty, random seed) |
| **dynamic_data** | struct | Runtime game state (tick count, player count, entity counts, object frequency tracking) |
| **static_data** | struct | Level metadata (environment code, physics model, song, mission, entry point flags) |
| **ambient_sound_image_data** | struct | Non-directional ambient sound (flags, index, volume) |
| **random_sound_image_data** | struct | Directional random sound (flags, volume delta, period delta, pitch delta, phase) |
| **object_frequency_definition** | struct | Monster/item spawn rules (initial/min/max count, random chance) |
| **saved_lighting_function_specification** | struct | Light animation parameters (function, period, intensity) |
| **saved_static_light_data** | struct | Static light with 6 animation specs (active/inactive/becoming) |
| **shape_and_transfer_mode** | struct | Extended shape descriptor for rendering (collection, shape index, transfer mode, animation frame/phase) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| **static_world** | static_data\* | global | Pointer to current level's static properties |
| **dynamic_world** | dynamic_data\* | global | Pointer to current level's runtime state and counts |
| **ObjectList** | vector<object_data> | global | All active game objects |
| **EndpointList** | vector<endpoint_data> | global | All map vertices |
| **LineList** | vector<line_data> | global | All map line segments |
| **SideList** | vector<side_data> | global | All map surfaces |
| **PolygonList** | vector<polygon_data> | global | All map polygons (floors/ceilings) |
| **AmbientSoundImageList** | vector<ambient_sound_image_data> | global | Ambient sound sources |
| **RandomSoundImageList** | vector<random_sound_image_data> | global | Random sound sources |
| **MapIndexList** | vector<int16> | global | Spatial index for object lookup |
| **AutomapLineList** | vector<uint8> | global | Bitmask of discovered lines (for automap) |
| **AutomapPolygonList** | vector<uint8> | global | Bitmask of discovered polygons (for automap) |
| **MapAnnotationList** | vector<map_annotation> | global | Text annotations on map |
| **SavedObjectList** | vector<map_object> | global | Initial object placements from map file |
| **game_is_networked** | bool | global | True if network multiplayer game |
| **LandscapesLoaded** | bool | global | Whether Marathon 2/oo landscape textures are active |
| **LoadedWallTexture** | short | global | Index of main wall texture (for infravision fog) |

## Key Functions / Methods

### initialize_marathon
- **Purpose**: Initialize the game engine and world system
- **Calls**: Various subsystem initializers
- **Notes**: Called once at startup

### entering_map / leaving_map
- **Purpose**: Setup/cleanup when transitioning between levels
- **Inputs**: entering_map takes `bool restoring_saved` (skip Pfhortran init on restore)
- **Side effects**: Allocates/frees map memory, initializes geometry and objects
- **Notes**: entering_map returns bool indicating success

### update_world
- **Purpose**: Main game loop tick; updates all entities, physics, and game state
- **Outputs/Return**: `pair<bool, int16>` ΓÇö (whether anything changed, elapsed real-mode ticks)
- **Side effects**: Increments tick counters, moves monsters, updates effects, handles input
- **Calls**: Monster and projectile updates, lighting updates, device updates
- **Notes**: Can trigger re-rendering even if no real ticks elapsed (for prediction)

### new_map_object / new_map_object2d / new_map_object3d
- **Purpose**: Create a new object at a given location
- **Inputs**: Location (2D or 3D), polygon index, shape descriptor, facing angle
- **Outputs/Return**: Object index (short), or NONE if failed
- **Side effects**: Adds object to ObjectList and object's home polygon; updates counts

### remove_map_object
- **Purpose**: Remove an object from the world
- **Inputs**: Object index
- **Side effects**: Removes from polygon object list, marks slot as free, updates counts

### translate_map_object
- **Purpose**: Move an object to a new location and polygon
- **Inputs**: Object index, new location, new polygon index
- **Outputs/Return**: bool (success)
- **Side effects**: Updates polygon object lists, object location and polygon fields

### remove_object_from_polygon_object_list / deferred_add_object_to_polygon_object_list / perform_deferred_polygon_object_list_manipulations
- **Purpose**: Manipulate polygon object lists with deferred execution (for network prediction)
- **Inputs**: Object index, optional target polygon; second function takes index_to_precede
- **Side effects**: Schedules insertions/removals; actual manipulation on deferred call
- **Notes**: Used by prediction system to avoid immediate map state changes

### world_point_to_polygon_index
- **Purpose**: Find which polygon contains a 2D point
- **Inputs**: Pointer to world_point2d
- **Outputs/Return**: Polygon index (short), or NONE if outside all polygons
- **Calls**: point_in_polygon for candidate polygons

### point_in_polygon
- **Purpose**: Test if a 2D point is inside a polygon
- **Inputs**: Polygon index, pointer to world_point2d
- **Outputs/Return**: bool

### find_adjacent_polygon
- **Purpose**: Find the polygon on the other side of a line
- **Inputs**: Current polygon index, line index
- **Outputs/Return**: Adjacent polygon index (short), or NONE if no adjacent

### find_line_intersection / find_floor_or_ceiling_intersection
- **Purpose**: Raycast tests for collision with map geometry
- **Inputs**: Line endpoints or height and endpoints
- **Outputs/Return**: Fixed-point parameter along ray [0, FIXED_ONE]; stores intersection point
- **Notes**: Used for line-of-sight, projectile collision

### keep_line_segment_out_of_walls
- **Purpose**: Enforce maximum height (ceiling) constraint for movement
- **Inputs**: Polygon, start/end points, max height change, current height
- **Outputs/Return**: bool; fills adjusted floor/ceiling heights and supporting polygon
- **Side effects**: None
- **Notes**: Used by monster/player movement code

### point_is_player_visible / point_is_monster_visible
- **Purpose**: Test visibility from player/monster perspective
- **Inputs**: Point location, polygon, optional max player index for player version
- **Outputs/Return**: bool; fills distance if visible
- **Side effects**: None (used for AI and rendering)

### play_object_sound / play_polygon_sound / play_world_sound
- **Purpose**: Queue audio to play from object or location
- **Inputs**: Object/polygon/side index or 3D origin, sound code
- **Signature** (play_side_sound): `#define play_side_sound(side_index, sound_code) _play_side_sound(side_index, sound_code, FIXED_ONE)`
- **Side effects**: Queues audio event; pitch defaults to 1.0 for side sounds

### calculate_damage
- **Purpose**: Compute actual damage amount from a damage_definition
- **Inputs**: Pointer to damage_definition
- **Outputs/Return**: Damage amount (short)
- **Notes**: Applies random variance and difficulty scaling

### cause_polygon_damage
- **Purpose**: Apply damage to entities in a polygon
- **Inputs**: Polygon index, monster index (source)
- **Side effects**: Damages monsters/player in polygon, triggers events
- **Calls**: calculate_damage, entity damage handlers

### change_polygon_height
- **Purpose**: Change floor/ceiling height of a polygon (platform movement)
- **Inputs**: Polygon index, new heights, damage_definition
- **Outputs/Return**: bool (success)
- **Side effects**: Updates polygon, applies damage to crushed entities, triggers adjacent updates
- **Notes**: Can trigger object relocation if entities move between polygons

### animate_object
- **Purpose**: Advance object animation by one tick
- **Inputs**: Object index
- **Side effects**: Updates object.sequence frame/phase; sets animation flags

### get_object_shape_and_transfer_mode / set_object_shape_and_transfer_mode
- **Purpose**: Get/set rendering properties with animation frame info
- **Inputs**: Camera location (for get), object index, shape descriptor, transfer mode
- **Outputs**: Populates shape_and_transfer_mode struct with frame/phase data
- **Notes**: get variant used by renderer to determine correct animated frame

### changed_polygon
- **Purpose**: Notify system that player/entity moved to a new polygon
- **Inputs**: Original polygon, new polygon, player index
- **Side effects**: Updates lights, activates triggers, updates ambient sounds
- **Calls**: Light state changes, device activations, sound updates

### generate_map
- **Purpose**: Load and build a level
- **Inputs**: Level index
- **Side effects**: Populates all map structures (geometry, objects, sounds, lights)
- **Notes**: Called during level load

### new_game / goto_level
- **Purpose**: Start a new game or advance to a level
- **Inputs**: Player count, network flag, game_data, player_start_data array, entry point
- **Outputs/Return**: bool (success)
- **Side effects**: Initializes game state, spawns players, resets game clock

**Notes on trivial helpers**: The file also exports many utility functions for geometric calculations (distance_squared, closest_point_on_line, ray_to_line_segment, etc.) used by physics and pathfinding code. Data accessor functions (get_object_data, get_polygon_data, etc.) are de-inlined for code size; serialization pack/unpack functions handle byte-stream conversion for save/network.

## Control Flow Notes

**Initialization Chain**:
1. `initialize_marathon()` ΓåÆ Sets up engine
2. `new_game()` ΓåÆ Create game with players/options
3. `goto_level()` or `entering_map()` ΓåÆ Load level
4. `generate_map()` ΓåÆ Build geometry from map file
5. `initialize_map_for_new_level()` ΓåÆ Setup level state

**Per-Frame**:
- `update_world()` ΓåÆ Main tick (updates monsters, projectiles, effects, lighting, device state)
- May call `changed_polygon()` when player crosses lines
- Checks visibility with `point_is_player_visible()` / `point_is_monster_visible()`

**Object Lifecycle**:
- `new_map_object*()` ΓåÆ Create in ObjectList and polygon
- `translate_map_object()` ΓåÆ Move between polygons
- Monsters/projectiles update their own animation via `animate_object()`
- Physics code queries geometry via `world_point_to_polygon_index()`, `find_line_intersection()`, etc.
- `remove_map_object()` ΓåÆ Cleanup

**Network Prediction**:
- `set_prediction_wanted()` / `reset_intermediate_action_queues()` ΓåÆ Manage prediction state
- Uses deferred operations (`deferred_add_object_to_polygon_object_list()`) to avoid immediate state changes

## External Dependencies

- **csmacros.h**: Macro utilities (FLAG, TEST_FLAG, SET_FLAG, MIN, MAX, PIN, ABS)
- **world.h**: World coordinate system (world_distance, world_point2d/3d, angle, fixed-point math, trig tables)
- **dynamic_limits.h**: Runtime entity limits via `get_dynamic_limit()`
- **XML_ElementParser.h**: XML parsing for texture-loading config
- **shape_descriptors.h**: Shape/collection/texture descriptor bit-packing macros
- **<vector>**: STL dynamic arrays for map structures

**Defined Elsewhere** (declared extern):
- Trigonometric tables (cosine_table, sine_table from world.h)
- Damage handlers (monster/player damage applications)
- Rendering system (shape animation, transfer mode rendering)
- Monster/projectile/effect managers (driven by update_world)
- Device/lighting/sound systems (called by changed_polygon and update_world)
