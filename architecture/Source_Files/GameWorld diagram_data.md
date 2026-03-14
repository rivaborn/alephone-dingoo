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

# Source_Files/GameWorld/dynamic_limits.cpp
## File Purpose
Manages configurable runtime limits for game entities (objects, monsters, projectiles, effects, paths). Allows limits to be loaded from XML configuration instead of being hardcoded, with fallback to reasonable defaults and support for resetting to original values.

## Core Responsibilities
- Maintains array of 8 dynamic limit values with defaults higher than originals
- Backs up and restores original limit values when configuration is reset
- Parses XML elements to configure individual limits with validation (0ΓÇô32767 range)
- Coordinates resizing of game entity container vectors when limits change
- Allocates pathfinding memory proportionally to configured path limits
- Provides accessor function for other modules to query current limits by type

## External Dependencies
- **`cseries.h`**: Provides macros, assertions, and utility functions.
- **`dynamic_limits.h`**: Enum definitions (`_dynamic_limit_*`), `NUMBER_OF_DYNAMIC_LIMITS`, parser interface.
- **`map.h`**: Declares `MAXIMUM_OBJECTS_PER_MAP` macro using `get_dynamic_limit()`.
- **`effects.h`**: Declares `MAXIMUM_EFFECTS_PER_MAP`; vector `EffectList`.
- **`monsters.h`**: Declares `MAXIMUM_MONSTERS_PER_MAP`; vector `MonsterList`; collision buffer size macros.
- **`projectiles.h`**: Declares `MAXIMUM_PROJECTILES_PER_MAP`; vector `ProjectileList`.
- **`flood_map.h`**: Header only; declares `allocate_pathfinding_memory()` (called from `End()`).
- **`XML_ElementParser` (base class)**: Provides XML parsing framework; defined elsewhere.
- **Utility functions** (`StringsEqual`, `ReadBoundedUInt16Value`, `UnrecognizedTag`, `AttribsMissing`): Defined elsewhere in CSeries.

# Source_Files/GameWorld/dynamic_limits.h
## File Purpose
Defines the interface for querying and configuring dynamic runtime limits on game entities (objects, monsters, projectiles, effects, etc.). Supports XML-based configuration loading instead of resource forks. Provides centralized access to entity count constraints that affect AI pathfinding, rendering, and collision systems.

## Core Responsibilities
- Enumerate all dynamic limit types (objects, NPCs, projectiles, effects, collision buffers, rendering queue)
- Declare XML parser factory for loading limits from configuration files
- Provide runtime accessor for querying specific entity count limits
- Support both local and global collision buffer capacity constraints

## External Dependencies
- **XML_ElementParser.h** ΓÇö Base class for XML parsing infrastructure
- **cstypes.h** ΓÇö Type definitions (uint16, int)
- **"COPYING"** ΓÇö GPL license file reference

# Source_Files/GameWorld/editor.h
## File Purpose
Header file for the map editor subsystem defining data structures, version constants, and geometric bounds. Establishes compatibility across Marathon 1/2/Infinity data formats and constrains valid map geometry.

## Core Responsibilities
- Define data version constants for Marathon series format compatibility
- Specify map coordinate bounds (min/max X/Y)
- Specify floor/ceiling height constraints with safety margins
- Define guard path control point structures (`saved_path`)
- Define map metadata structure (`map_index_data`)
- Define validation constants (max control points, lines per vertex)

## External Dependencies
- **Defines used but not in this file:**
  - `WORLD_ONE` ΓÇö engine unit scale constant
  - `SHORT_MIN`, `SHORT_MAX` ΓÇö C type limits
  - `LEVEL_NAME_LENGTH` ΓÇö array size for level names
  - `world_point2d` ΓÇö 2D coordinate struct (likely defined elsewhere)

- **Version strategy:** `EDITOR_MAP_VERSION` is set to `MARATHON_INFINITY_DATA_VERSION` (value 2), indicating the editor targets Marathon Infinity format as the canonical interchange format across all three game versions.

# Source_Files/GameWorld/effect_definitions.h
## File Purpose
Defines all visual and audio effects for the game world, including weapon detonations, blood splashes, media interactions, and environmental effects. Provides configuration data structures and default effect definitions, plus serialization utilities for save/load.

## Core Responsibilities
- Declares `effect_definition` struct to configure animation source, audio pitch, and timing behavior for each effect type
- Defines flags controlling effect termination conditions, audio-only modes, and media interaction
- Initializes `original_effect_definitions[]` array mapping ~60+ effect enum IDs to animation/sound configurations
- Manages runtime copy `effect_definitions[]` for potential modifications
- Exports pack/unpack functions for serializing effect data to/from streams

## External Dependencies
- **Collection constants** (e.g., `_collection_rocket`, `_collection_fighter`, `_collection_cyborg`): defined elsewhere, identify animation sprite sheets
- **Frequency constants** (e.g., `_normal_frequency`, `_higher_frequency`, `_lower_frequency`): pitch multipliers for sound playback
- **Sound constants** (e.g., `_snd_teleport_in`, `NONE`): sound IDs to play with effect
- **Macro** `BUILD_COLLECTION()`: packs collection variant; likely used for alternate art assets
- **Constants** `TICKS_PER_SECOND`, `NUMBER_OF_EFFECT_TYPES`: game timing/bounds

**Note**: Header guard typo: `__EFFECT_DEFINTIIONS_H` should be `__EFFECT_DEFINITIONS_H` (misspelled "DEFINITIONS").

# Source_Files/GameWorld/effects.cpp
## File Purpose
Implements the runtime effect system for the game engine. Effects are temporary visual and audio phenomena (explosions, splashes, contrails, teleportation) spawned at world locations and animated until completion. This file manages effect instance creation, per-frame updates, removal, and persistence state tracking.

## Core Responsibilities
- Create and track active effect instances at world locations
- Update effect animations and detect termination conditions
- Manage visibility and sound playback for delayed/invisible effects
- Handle specialized teleportation effects that coordinate with game objects
- Serialize/deserialize effect data for save/load
- Mark required shape collections for resource loading

## External Dependencies
- **map.h** ΓÇö world geometry, object management, animation macros, slot-in-use macros
- **interface.h** ΓÇö shape animation metadata (`get_shape_animation_data`)
- **SoundManager.h** ΓÇö `play_world_sound`, `play_object_sound`
- **Packing.h** ΓÇö stream serialization utilities
- **effect_definitions.h** ΓÇö effect type enums, flags, default definition array

# Source_Files/GameWorld/effects.h
## File Purpose
Header for the game engine's visual effects system. Defines the interface for creating, managing, and updating game effects (explosions, blood splatters, splashes, contrails, etc.) that occur during gameplay. Effects are temporary or persistent visual phenomena tied to world events.

## Core Responsibilities
- Define 70+ effect types (explosions, contrails, blood splashes, liquid splashes, detonations, etc.)
- Manage active effect data structure and list
- Provide creation/destruction interface for effects
- Handle per-tick effect updates
- Manage effect serialization/deserialization for save/load
- Track dynamic limits on active effects per map
- Provide teleport effect operations

## External Dependencies
- **Includes:** `dynamic_limits.h` (for `get_dynamic_limit()` macro call in `MAXIMUM_EFFECTS_PER_MAP`).
- **Types used:** `world_point3d`, `angle` (defined elsewhere, likely in geometry headers).
- **External symbols:** `get_dynamic_limit(_dynamic_limit_effects)` ΓÇô retrieves current effect limit.
- **Language features:** Uses C++ `vector<>` (modern C++ container, not pure C).

# Source_Files/GameWorld/flood_map.cpp
## File Purpose
Implements a stateful graph search algorithm for pathfinding across polygon-based game world geometry. Supports multiple search strategies (breadth-first, best-first, depth-first) with customizable cost evaluation and node selection.

## Core Responsibilities
- Manage dynamic node allocation and search tree construction for polygon traversal
- Execute multiple flood-fill search algorithms (best-first, breadth-first, flagged breadth-first)
- Track visited polygons to prevent revisits and enable path reconstruction
- Support reverse path traversal (backtracking from destination to source)
- Provide depth calculation and biased random node selection for pathfinding algorithms

## External Dependencies
- **Includes:**
  - `cseries.h`: Utility macros and types
  - `map.h`: World geometry (polygon_data, endpoints, lines, etc.)
  - Standard C: `string.h`, `stdlib.h`, `limits.h`
- **Functions defined elsewhere:**
  - `get_polygon_data(int16)`: Accessor from map.h
  - `find_center_of_polygon(int16, world_point2d*)`: Utility from map.h
  - `global_random()`: RNG (defined elsewhere)
  - `objlist_set()`: Array initialization utility
- **Global data:**
  - `dynamic_world`: World state containing polygon_count and other runtime data

# Source_Files/GameWorld/flood_map.h
## File Purpose
Header file declaring the flood-map pathfinding system for the Aleph One game engine. Provides interfaces for computing navigable paths between map polygons using multiple flood-fill traversal strategies and custom cost functions.

## Core Responsibilities
- Define flood-fill traversal modes (depth-first, breadth-first, flagged breadth-first, best-first)
- Declare pathfinding memory allocation and initialization
- Provide path creation/deletion/traversal operations
- Declare flood-map computation, reversal, and depth queries
- Support random node selection from flood results

## External Dependencies
- **Type imports (defined elsewhere):** `int32`, `short`, `bool`, `world_point2d`, `world_distance`, `world_vector2d`
- **Implicit dependencies:** Polygon/line index system (world topology model)

# Source_Files/GameWorld/item_definitions.h
## File Purpose
Defines the item metadata table for the game engine. This header declares the structure for item properties (weapons, ammunition, powerups, keys, balls) and provides a static lookup table mapping item kinds to their game properties and resource identifiers.

## Core Responsibilities
- Define the `item_definition` structure for item metadata
- Declare a static global array of all item definitions indexed by item kind
- Specify weapon/ammo/powerup properties: singular/plural names, shape descriptors, carry limits, and environmental restrictions
- Serve as the authoritative reference for item capabilities used by game systems

## External Dependencies
- **Macros**: `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()` ΓÇö construct shape/collection identifiers (defined elsewhere)
- **Enum constants**: `_weapon`, `_ammunition`, `_powerup`, `_item`, `_ball` ΓÇö item kind types
- **Environment flags**: `_environment_vacuum`, `_environment_single_player` ΓÇö restrict items to certain world contexts
- **Magic constants**: `UNONE`, `NONE` ΓÇö likely "undefined" or "no shape" sentinels
- **Naming**: String table IDs (e.g., `0`, `1`, `2`) ΓÇö resolved at runtime to display names

**Not inferable from this file:**
- The definition of `shape_descriptor` type
- How `item_definitions` is actually indexed or iterated by other systems
- Behavior when `invalid_environments` flags prevent pickup

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

## External Dependencies

- **Geometry/world**: `map.h` (polygons, objects, lines, platforms)
- **Entities**: `monsters.h`, `player.h` (entity data, inventory)
- **Systems**: `weapons.h` (reload), `SoundManager.h` (audio), `fades.h` (screen effects)
- **State**: `network_games.h` (multiplayer detection), `lua_script.h` (scripting callbacks)
- **Global**: `item_definitions` array, `dynamic_world`, `static_world`, `objects` array, game environment flags

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

## External Dependencies
- **XML_ElementParser.h** ΓÇö XML parsing infrastructure for item configuration
- **cstypes.h** (via XML_ElementParser.h) ΓÇö Custom type definitions
- **Defined elsewhere:**  
  - `struct object_location` ΓÇö World position/placement data  
  - `struct item_definition` ΓÇö Item configuration (shape, behavior, properties)  
  - Item enums and functions implemented in items.c

# Source_Files/GameWorld/lightsource.cpp
## File Purpose

Implements the dynamic lighting system for the Aleph One game engine (Marathon series). Manages light source creation, state transitions, intensity animation, and serialization. Lights cycle through states (becoming active ΓåÆ primary active ΓåÆ secondary active, and the inverse for inactive), with intensity computed by pluggable animation functions (constant, linear, smooth, flicker).

## Core Responsibilities

- **Light lifecycle**: Create new lights (`new_light`), track active lights in global `LightList`, support cleanup via slot marking
- **Frame updates**: Advance light phases each tick, trigger state transitions when phase overflows period, recalculate intensity
- **State management**: Transition lights between 6 states (becoming/primary/secondary ├ù active/inactive); support stateless lights that skip intermediate transitions
- **Intensity animation**: Dispatch 4 lighting functions (constant, linear, smooth with cosine taper, flicker with randomness)
- **Status control**: Query and toggle light on/off status; support bulk switching by tag; integrate with Lua script hooks
- **Serialization**: Pack/unpack light data to/from binary streams; convert Marathon 1 light format to Marathon 2+ format
- **Default configurations**: Define 3 standard light types (_normal_light, _strobe_light, _lava_light) with preset animation specs

## External Dependencies

- **cseries.h** ΓÇö `vhalt`, `csprintf`, `temporary`, assertion macros, common types.
- **map.h** ΓÇö `TICKS_PER_SECOND`, `MAXIMUM_LIGHTS_PER_MAP`, `LIGHT_IS_INITIALLY_ACTIVE`, `SLOT_IS_USED/FREE/MARK_*` macros, world definitions, `assume_correct_switch_position`.
- **lightsource.h** ΓÇö Type definitions (`light_data`, `static_light_data`, `lighting_function_specification`, etc.), enum constants, global `LightList`.
- **Packing.h** ΓÇö `StreamToValue`, `ValueToStream` for binary serialization.
- **lua_script.h** ΓÇö `L_Call_Light_Activated()` callback (only when `HAVE_LUA` is defined).
- **Global functions (defined elsewhere):** `global_random()`, `cosine_table[]`, `GetMemberWithBounds()`, `obj_copy()`, `SET_FLAG`, `TEST_FLAG16`, `SET_FLAG16`.

# Source_Files/GameWorld/lightsource.h
## File Purpose
Defines the light source system for the Marathon game engine, including data structures for static light configuration and dynamic light state. Implements a state-machine-based lighting model with multiple transition functions and provides backwards compatibility with Marathon I light format.

## Core Responsibilities
- Define light data structures: static configuration (`static_light_data`), dynamic state (`light_data`), and legacy format (`old_light_data`)
- Specify lighting function types (constant, linear, smooth, flicker) for intensity transitions
- Manage a global light list and provide creation/query/control operations
- Support serialization and deserialization of light data (for save/load and network)
- Convert legacy Marathon I light format to Marathon II format for map compatibility

## External Dependencies
- `#include <vector>` ΓÇö C++ standard library for dynamic light collection
- Custom macros: `TEST_FLAG16()`, `SET_FLAG16()` (flag manipulation, defined elsewhere)
- Custom type: `_fixed` (likely fixed-point arithmetic type, defined elsewhere)
- Serialization functions suggest a structured binary format shared with map/save systems

# Source_Files/GameWorld/map.cpp
## File Purpose
Core map and world management for the Marathon engine. Manages game world geometry (polygons, lines, endpoints, sides), object lifecycle (creation/removal/updates), collision detection, visibility calculations, and sound propagation. Also handles dynamic texture collection loading and map initialization.

## Core Responsibilities
- Allocate and initialize world data structures (`static_world`, `dynamic_world`) and dynamic lists (objects, monsters, projectiles, effects, etc.)
- Create, update, and destroy map objects with proper polygon linking
- Manage polygon-to-object associations via linked lists for efficient spatial queries
- Detect collision and obstruction (line-crossing, visibility, sound propagation)
- Load/unload texture collections based on environment and visible geometry
- Animate objects and generate random spatial points
- Play and manage world sounds with obstruction and media filtering
- Parse XML configuration for texture environment mapping
- Handle garbage object limits and corpse cleanup

## External Dependencies
- **cseries.h** ΓÇö common series utilities, macros, types (obj_clear, assert, etc.)
- **map.h** ΓÇö own header with structure defs and function declarations
- **interface.h** ΓÇö collection/shape management (mark_collection_for_loading, etc.)
- **monsters.h** ΓÇö monster data access, definitions (defined elsewhere)
- **preferences.h** ΓÇö graphics preferences (corpse limits)
- **projectiles.h** ΓÇö projectile data access
- **effects.h** ΓÇö effect data and creation
- **player.h** ΓÇö player data and creation
- **platforms.h** ΓÇö platform interaction
- **lightsource.h** ΓÇö light source management
- **lua_script.h** ΓÇö Lua scripting hooks
- **media.h** ΓÇö media (water/lava) data and queries
- **scenery.h** ΓÇö scenery object metadata
- **SoundManager.h** ΓÇö sound playback system
- **Console.h** ΓÇö console/scripting interface
- **XML_ElementParser.h** ΓÇö XML configuration parsing
- **Standard C/C++** ΓÇö string.h, stdlib.h, limits.h, `<list>`

**Symbols defined elsewhere (called/used here):**
- `GetMemberWithBounds()` ΓÇö bounds-checked vector access
- `objlist_clear()` ΓÇö clear object list
- `possible_intersecting_monsters()` ΓÇö collision detection
- `get_media_data()`, `get_platform_data()`, etc. ΓÇö accessors (defined in other files or inlined in map.h)
- `line_transfer_mode_is_smeared()`, `line_transfer_mode_stops_line_of_sight()` ΓÇö line property queries
- `change_light_state()`, `entered_polygon()`, `left_polygon()` ΓÇö external callbacks for polygon events
- `SoundManager::instance()->PlaySound()` ΓÇö global sound system
- Global player/monster/projectile arrays/lists ΓÇö accessed via macros and functions

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

# Source_Files/GameWorld/map_constructors.cpp
## File Purpose

Core map geometry construction and precalculation system for the Aleph One engine (Marathon). Handles creation of polygons, sides, lines, and endpoints; calculates geometric properties (area, adjacencies, exclusion zones); and implements binary packing/unpacking for save file serialization.

## Core Responsibilities

- Create new map geometry primitives (sides, polygons, lines, endpoints)
- Calculate and cache geometric properties: polygon area, clockwise endpoint ordering, adjacent polygons/sides
- Classify side types (_full_side, _high_side, _low_side, _split_side) based on adjacent polygon heights
- Precalculate spatial indexes for collision detection (exclusion zones, neighbor lists)
- Assign lightsources to sides based on side type and polygon lighting
- Flood-fill algorithm to find intersecting endpoints/lines/polygons within a separation distance
- Serialize/deserialize all map data structures to/from binary streams (big-endian format)
- Manage dynamic map index buffer allocation

## External Dependencies

- **map.h** ΓÇô map data structure definitions (polygon_data, line_data, side_data, endpoint_data, object_data, dynamic_world, static_world, SideList, PolygonList, etc.)
- **flood_map.h** ΓÇô `flood_map()` function for spatial traversal
- **platforms.h** ΓÇô `platform_data`, `get_platform_data()` for platform-specific height queries
- **cseries.h** ΓÇô macros and utilities (obj_clear, TEST_FLAG, SET_FLAG, etc.)
- **Packing.h** ΓÇô `StreamToValue()`, `ValueToStream()`, `StreamToList()`, `ListToStream()`, `StreamToBytes()`, `BytesToStream()` macros for binary serialization
- **limits.h** ΓÇô INT16_MIN, INT16_MAX, INT32_MAX
- **vector** (STL) ΓÇô `vector<short>` containers

Notable symbols used but not defined here: `find_center_of_polygon()`, `distance2d()`, `push_out_line()`, `clockwise_endpoint_in_line()`, `find_adjacent_polygon()`, `flood_map()`, `get_platform_data()`, `precalculate_polygon_sound_sources()`, `add_map_index()`, various accessor macros.

# Source_Files/GameWorld/marathon2.cpp
## File Purpose
Core game world orchestration for the Marathon engine, implementing the main game loop, world updates, and level lifecycle. Handles player action queue management and supports predictive client-side movement for networking.

## Core Responsibilities
- World initialization and shutdown
- Main game tick update loop with per-frame simulation
- Level loading/unloading and transitions
- Player action queue processing (real, Lua, and predictive)
- Predictive movement simulation for network lag compensation
- Polygon trigger activation (lights, platforms, monsters, items)
- Damage calculation and level completion state checking
- Game-over and level-change detection

## External Dependencies
- **Core Game Subsystems:** `map.h` (geometry), `player.h` (player state), `monsters.h`, `projectiles.h`, `effects.h`, `weapons.h`, `items.h`, `platforms.h`, `lightsource.h`, `media.h` (fluids), `scenery.h`
- **Rendering & UI:** `render.h`, `interface.h`, `game_window.h`, `fades.h`
- **Audio:** `Music.h`, `SoundManager.h`
- **Network:** `network.h`, `network_games.h` (game sync, chat callbacks)
- **Scripting:** `lua_script.h`, `lua_hud_script.h` (Lua integration, action queue overlay)
- **Support:** `ChaseCam.h`, `OGL_Setup.h`, `AnimatedTextures.h`, `tags.h`, `Console.h`, `ActionQueues.h` (action flag queues), `Logging.h`

---

**Notes on Architecture:**
- The file is the primary orchestrator of game-world simulation, delegating subsystem updates to specialized modules.
- Prediction system decouples client-side responsiveness from server-authoritative state, critical for network play.
- Action queue overlay design permits Lua scripts and network events to override real input without reimplementing the update loop.
- Level transitions are asynchronous via state flags (`check_level_change()`) rather than exceptions, keeping simulation linear.

# Source_Files/GameWorld/media.cpp
## File Purpose
Manages in-game media (liquids: water, lava, goo, sewage, Jjaro) for the Aleph One game engine. Handles media instance lifecycle, per-frame updates (height/texture/movement), property queries (damage, sounds, effects), and XML-driven customization of media definitions.

## Core Responsibilities
- Create, store, and retrieve media instances on the current map
- Update media states each frame (height derived from light intensity, texture, movement via current)
- Query media properties (detonation effects, ambient/event sounds, damage, submerged fade effects, texture collections)
- Parse and apply XML overrides to default media type definitions
- Serialize/deserialize media data to/from packed byte streams

## External Dependencies
- **Includes:** map.h (media_data limits, macros), effects.h, fades.h, lightsource.h, SoundManager.h, DamageParser.h, Packing.h, XML_ElementParser.h.
- **Defined elsewhere:** media_definitions (array), light intensity lookup, damage types, effect/sound/fade enums, `GetMemberWithBounds()`, XML parser base class.
- **Macros used:** `SLOT_IS_USED()`, `MARK_SLOT_AS_USED()`, `CALCULATE_MEDIA_HEIGHT()`, `WORLD_FRACTIONAL_PART()`, `BUILD_DESCRIPTOR()`, trig lookup tables.

# Source_Files/GameWorld/media.h
## File Purpose
Header for the liquid media system (water, lava, goo, sewage, Jjaro). Defines media types, properties, and the central `media_data` struct; provides factory, query, and serialization functions for managing all fluids in a game level.

## Core Responsibilities
- Define media type constants and flags (water, lava, goo, sewage, jjaro)
- Define detonation effects and ambient/transition sounds for media interactions
- Declare the `media_data` structure (32 bytes) holding media properties: type, flags, height, current, texture, light binding
- Provide factory and update functions (`new_media`, `update_medias`)
- Query media properties: damage, sound effects, submerged rendering, danger level, environment compatibility
- Serialize/deserialize media data to/from byte streams (for save games)
- Expose XML parser integration for map loading

## External Dependencies
- `#include <vector>` ΓÇö STL container for dynamic media list
- `#include "map.h"` ΓÇö Provides `SLOT_IS_USED` macro and world coordinate types
- `#include "XML_ElementParser.h"` ΓÇö Base class for XML parsing integration

# Source_Files/GameWorld/media_definitions.h
## File Purpose
Defines static configuration data for all in-game media types (water, lava, goo, sewage, Jjaro). Specifies visual representation, damage properties, sound effects, and detonation effects for each medium the player can interact with.

## Core Responsibilities
- Define the `media_definition` struct schema (visual, audio, damage, effects properties)
- Initialize a global static array with configurations for 5 distinct media types
- Specify splash/detonation effects for small/medium/large interactions with each medium
- Configure ambient and interaction sound effects for each medium
- Define damage rules (frequency and type) when submerged in each medium
- Map each medium to its graphical representation (collection/shape)

## External Dependencies
- External enums/constants: `_media_water`, `_collection_walls1ΓÇô5`, `_xfer_normal`, `_effect_*` (splash/emergence), `_snd_*` (sound IDs), `_damage_lava`, `_damage_goo`, etc.
- External struct: `damage_definition`
- Constants: `NONE`, `FIXED_ONE`, `NUMBER_OF_MEDIA_TYPES`, `NUMBER_OF_MEDIA_DETONATION_TYPES`, `NUMBER_OF_MEDIA_SOUNDS`

All symbols defined elsewhere. Not inferable from this file.

# Source_Files/GameWorld/monster_definitions.h
## File Purpose

Defines all monster types and their combat/behavioral properties in the Marathon/Aleph One game engine. Contains complete specifications for 30+ monster variants, including statistics (health, speed, attack patterns), class relationships (friend/foe), and rendering/sound data. Serves as the central configuration for all AI-controlled creatures.

## Core Responsibilities

- Define monster class hierarchy and allegiance relationships (Pfhor aliens, Compilers, Defenders, Yetis, natives, player, humans)
- Specify combat parameters: vitality, attack types/ranges, projectile capabilities, speeds
- Configure visual representation: shape collections, animation frames, teleport effects, size variants (major/minor/tiny)
- Set sensory parameters: visual range, intelligence levels, sound effects
- Control environmental interactions: door-opening capability, ledge navigation, media-specific behaviors (lava/water/goo fears)
- Provide pack/unpack serialization for monster definitions

## External Dependencies

- **effects.h**: Effect type constants referenced in monster definitions (e.g., `_effect_fighter_blood_splash`, `_effect_juggernaut_spark`)
- **projectiles.h**: Projectile type constants for melee/ranged attacks (e.g., `_projectile_staff`, `_projectile_compiler_bolt_major`)
- **Macro constants** (defined elsewhere): `WORLD_ONE`, `FIXED_ONE`, `TICKS_PER_SECOND`, `NUMBER_OF_ANGLES`, `BUILD_COLLECTION`, `FLAG`, `NONE`, `UNONE`, `QUARTER_CIRCLE`
- **Type definitions** (from broader codebase): `int16`, `uint32`, `_fixed`, `angle`, `world_distance`, `shape_descriptor`, `damage_definition`

---


# Source_Files/GameWorld/monsters.cpp
## File Purpose
Implements monster AI, physics, animation, and lifecycle for the Marathon engine. Handles monster spawning, pathfinding, target acquisition, combat behavior, damage application, and serialization. Core to the enemy entity system.

## Core Responsibilities
- Monster creation with difficulty-based scaling (promotion/demotion) and map placement
- Frame-by-frame monster update loop: animation, pathfinding, target acquisition, physics, state transitions
- Pathfinding and movement with cost-based navigation and terrain interaction (platforms, doors, media)
- Target acquisition via line-of-sight checking and attitude determination (friendly/hostile/neutral)
- Behavioral modes (locked, losing lock, lost lock, unlocked) and action states (moving, attacking, dying, teleporting)
- Physics simulation including vertical motion, external velocity from impacts, and gravity
- Damage application, death handling, shrapnel effects, and monster-monster interaction
- Save/load serialization via pack/unpack functions
- Runtime XML configuration for damage kick (knockback) definitions

## External Dependencies
- **Map system:** `get_polygon_data()`, `get_line_data()`, `find_adjacent_polygon()`, `cause_polygon_damage()`, `find_new_object_polygon()`, `ray_to_line_segment()`, `flood_map()` (pathfinding).
- **Object/Rendering:** `get_object_data()`, `new_map_object()`, `remove_map_object()`, `animate_object()`, `GET_OBJECT_ANIMATION_FLAGS()`, `GET_SEQUENCE_FRAME()`.
- **Physics/Platforms:** `monster_can_enter_platform()`, `monster_can_leave_platform()`, `get_media_data()`, `IsMediaDangerous()`.
- **Projectiles:** `fire_projectile()`, `position_monster_projectile()` (custom positioning), `orphan_projectiles()`.
- **Effects:** `new_effect()`, `teleport_object_in/out()`.
- **Sound:** `SoundManager::instance()->LoadSound()`, `play_object_sound()`.
- **Lua:** `L_Invalidate_Monster()` (optional scripting hook).
- **Monster Definitions:** Included via `monster_definitions.h` (external static data).
- **XML Parsing:** `XML_ElementParser` base class, `XML_DamageKickParser` for config.

# Source_Files/GameWorld/monsters.h
## File Purpose
Header file defining the monster/NPC system for the Marathon game engine (Aleph One). Declares structures, enums, and function interfaces for managing all non-player entitiesΓÇötheir types, behaviors, state transitions, damage, and lifecycle.

## Core Responsibilities
- Define monster data structures (`monster_data`) and persistent state (type, vitality, flags, action, mode)
- Enumerate ~40 monster types (ticks, cyborgs, hunters, civilians, yeties, defenders, etc.)
- Manage monster creation, despawning, and frame-by-frame updates
- Provide activation/deactivation and state queries via flag macros
- Handle damage application, death events, and collision/pathfinding integration
- Support sound-based and trigger-based monster activation systems
- Enable serialization (packing/unpacking) and XML-based configuration loading

## External Dependencies
- **Includes:** `dynamic_limits.h` (runtime limits), `XML_ElementParser.h` (config parsing), `<vector>` (STL storage)
- **Defined elsewhere:** `world_point3d`, `world_distance`, `angle`, `object_location`, `damage_definition`, `monster_definition`, `object_data`

# Source_Files/GameWorld/pathfinding.cpp
## File Purpose

Implements pathfinding for non-player characters (monsters) in the game world. Maintains a dynamic pool of pre-allocated paths that entities traverse sequentially. Uses a flood-fill algorithm to find routes through the polygon adjacency graph.

## Core Responsibilities

- Allocate and manage a pool of path objects with configurable maximum count
- Create new paths via flood-fill search from source to destination polygon
- Support both deterministic (goal-directed) and random path generation
- Advance entities one waypoint at a time along their assigned path
- Calculate safe passage points (midpoints of polygon-boundary lines)
- Validate path integrity in DEBUG builds

## External Dependencies

- **`flood_map.h`:** `flood_map()`, `reverse_flood_map()`, `flood_depth()`, `choose_random_flood_node()` ΓÇö core graph search; `cost_proc_ptr` callback type
- **`map.h`:** `find_shared_line()`, `get_line_data()`, `get_endpoint_data()`, polygon/line/endpoint data accessors
- **`dynamic_limits.h`:** `get_dynamic_limit()` for runtime limit configuration
- **`cseries.h`:** Macros `obj_set()`, `objlist_copy()`, assertion/debug utilities
- **Built-in C headers:** `<string.h>`, `<stdlib.h>`, `<limits.h>`

# Source_Files/GameWorld/physics.cpp
## File Purpose
Core player physics simulation engine for Marathon. Manages position, velocity, rotation, collision detection, and environmental interactions (media, gravity, jumping). Implements a continuous lag-tolerant networked physics model with frame-by-frame deterministic updates.

## Core Responsibilities
- Initialize and update player physics state each frame
- Calculate player movement based on input action flags (forward, strafe, rotate, look)
- Simulate gravity, jumping, and external forces (acceleration, impacts)
- Resolve collisions with walls, platforms, and solid objects
- Track player position and orientation (heading, pitch, elevation)
- Manage media (liquid) interaction and buoyancy
- Update animation state (step phase/amplitude for bob)
- Shadow physics state into world objects and camera position
- Pack/unpack physics constants for network sync and save/load

## External Dependencies
- **Key includes:** `render.h` (view data), `map.h` (polygon/line/object queries), `player.h` (player_data, physics_variables), `interface.h` (game state), `monsters.h` (monster_data), `media.h` (media_data), `ChaseCam.h` (chase-cam state check)
- **Key externals:** `static_world` (map physics model, environment flags), `local_player` (current player), `physics_models` (from physics_models.h), trigonometry tables (`cosine_table[]`, `sine_table[]`), `get_physics_constants_for_model()` (also defined locally)
- **Collision/world functions:** `keep_line_segment_out_of_walls()`, `translate_map_object()`, `legal_player_move()`, `find_new_object_polygon()` (defined in map.c)
- **Helper macros:** `PLAYER_IS_DEAD()`, `WORLD_TO_FIXED()`, `FIXED_TO_WORLD()`, `PIN()`, `SGN()`, bit manipulation (`FIXED_FRACTIONAL_BITS`, angle normalization)

# Source_Files/GameWorld/physics_models.h
## File Purpose
Defines character physics models (walking, running) and parameters controlling velocity, acceleration, rotation, and collision dimensions. Provides serialization interfaces for physics constants data.

## Core Responsibilities
- Define physics model enumeration (_model_game_walking, _model_game_running)
- Declare physics_constants structure with all movement/collision parameters
- Store original hardcoded physics constants for both models
- Provide pack/unpack functions for physics constant serialization
- Declare initialization function for physics model system

## External Dependencies
- `_fixed` type (fixed-point arithmeticΓÇödefined elsewhere)
- `FIXED_ONE`, `QUARTER_CIRCLE` constants (defined elsewhere)
- Standard C types: `uint8`, `size_t`

# Source_Files/GameWorld/placement.cpp
## File Purpose

Manages object (monster and item) placement and spawning in the game world. Handles initial level population, periodic respawning based on frequency rules, player spawn point selection, and validation of spawn locations.

## Core Responsibilities

- Load serialized placement frequency data for monsters and items into memory
- Place initial monsters and items according to map-defined placement rules
- Periodically recreate monsters and items when populations fall below minimum thresholds
- Track live object counts and manage minimum/maximum spawn constraints
- Find valid spawn locations from predefined map points and random locations
- Select optimal player starting positions considering proximity to monsters and other players
- Mark and preload monster collections and sounds required by the current map

## External Dependencies

- **map.h**: `object_location`, `polygon_data`, `object_data`, `saved_objects`, `dynamic_world`, geometry functions
- **monsters.h**: `new_monster()`, `activate_monster()`, `find_closest_appropriate_target()`, monster type enums
- **items.h**: `new_item()`, item type enums
- **cseries.h**: `global_random()`, memory utilities
- **map.c/world.c** (defined elsewhere): `point_is_player_visible()`, `point_is_monster_visible()`, `find_center_of_polygon()`, `find_new_object_polygon()`, `get_polygon_data()`, `get_object_data()`, `GET_GAME_OPTIONS()`, `get_player_starting_location_and_facing()`

# Source_Files/GameWorld/platform_definitions.h
## File Purpose
Defines platform type configurations for the Aleph One game engine. Contains a lookup table of platform definitions (doors, platforms, etc.) with their associated sounds, flags, and damage properties. This is a data-driven initialization file used to configure all platform behaviors in the game world.

## Core Responsibilities
- Define sound event enum for platform audio callbacks
- Specify the `platform_definition` structure that aggregates all properties for a platform type
- Initialize a global array of platform definitions for all platform types (8 types: SPHT doors, split doors, locked doors, platforms, heavy doors, Pfhor doors/platforms, etc.)
- Associate each platform type with its default configuration flags, damage model, and audio assets

## External Dependencies
- **Undefined enums**: Sound IDs (`_snd_spht_door_opening`, `_ambient_snd_spht_door`, etc.), platform flags (`_platform_is_spht_door`, `_platform_extends_floor_to_ceiling`, etc.), platform speeds (`_slow_platform`, `_fast_platform`), delay types (`_long_delay_platform`, `_very_long_delay_platform`), damage types (`_damage_crushing`)
- **Undefined types**: `static_platform_data`, `damage_definition`
- **Undefined macros**: `FLAG()`, `NONE`, `FIXED_ONE`, `NUMBER_OF_PLATFORM_TYPES`
- **Header guard**: `__PLATFORM_DEFINITIONS_H`

# Source_Files/GameWorld/platforms.cpp
## File Purpose

Implements the platform system for the Aleph One game engine, managing movable platforms (doors, elevators, platforms) in the game world. Handles platform physics, state transitions, player/monster interactions, geometry updates, and serialization.

## Core Responsibilities

- Create and initialize platforms from static configuration data
- Update all platforms each game tick (movement, obstruction handling, sound playback)
- Manage platform state activation/deactivation with cascading effects to adjacent platforms
- Detect obstruction when platforms move and handle reversal or blocking
- Adjust world geometry (polygon and endpoint heights) when platforms move
- Handle player and monster interactions (entry detection, state changes, key requirements)
- Evaluate platform accessibility for monster pathfinding
- Track media submersion state (floor/ceiling relative to liquids)
- Support XML-based runtime configuration of platform definitions
- Serialize/deserialize platform data for save games and network transmission

## External Dependencies

- **world.h**: `world_distance`, `world_point2d`, angle, trigonometry
- **map.h**: `polygon_data`, `endpoint_data`, `line_data`, `side_data`, `object_data`, world geometry accessors, change_polygon_height(), remove_map_object(), play_polygon_sound(), assume_correct_switch_position()
- **platforms.h**: Platform macros (FLAG tests/sets), `static_platform_data`, `platform_data`, `platform_definition`
- **lightsource.h**: `set_light_status()`
- **SoundManager.h**: `SoundManager::instance()`, play_polygon_sound() 
- **player.h**: `try_and_subtract_player_item()`
- **media.h**: `get_media_data()`
- **items.h**: (item definitions, used indirectly)
- **Packing.h**: `StreamToValue()`, `ValueToStream()` (binary serialization)
- **DamageParser.h**: `Damage_GetParser()`, `Damage_SetPointer()`
- **lua_script.h**: `L_Call_Platform_Activated()` (conditional on `HAVE_LUA`)

Notable: Relies heavily on bitflag macros from platforms.h for state management; uses GetMemberWithBounds for safe indexing; no explicit memory allocation beyond vector growth.

# Source_Files/GameWorld/platforms.h
## File Purpose
Defines platform (moving structure) types, state flags, and data structures for the game world. Provides the interface for creating, updating, and querying platformsΓÇöinteractive geometry like doors, elevators, and crushers that move between discrete positions.

## Core Responsibilities
- Enumerate platform types (doors, platforms with various behaviors)
- Define speed/delay presets for platform movement
- Define static (configuration) and dynamic (runtime) flag sets
- Provide flag testing/setting macros for platform properties
- Define platform data structures for both static config and runtime state
- Declare functions for platform creation, state management, and physics queries
- Provide serialization/deserialization for platform data
- Export XML parser interface for level loading

## External Dependencies
- **XML_ElementParser.h**: Base class for XML parsing; used for level file loading
- **Macro dependencies** (defined elsewhere): `TEST_FLAG16`, `TEST_FLAG32`, `SET_FLAG16`, `SET_FLAG32`, `WORLD_ONE`, `TICKS_PER_SECOND`, `MAXIMUM_VERTICES_PER_POLYGON`
- **Type dependencies** (defined elsewhere): `world_distance`, `int16`, `uint16`, `uint32`, `uint8`
- **Function dependencies**: Flag macros call into a flag-manipulation library (not visible here)

# Source_Files/GameWorld/player.cpp
## File Purpose
Core player entity implementation for the Marathon game engine. Manages player creation, state updates, damage/healing, weapon/powerup inventories, oxygen mechanics, death/respawn, and network synchronization across game ticks.

## Core Responsibilities
- **Player lifecycle**: Creation, initialization, deletion, and revival
- **Per-tick updates**: Physics integration, action processing, powerup decay, oxygen management
- **Damage system**: Calculate and apply damage, track damage given/taken per player/team
- **Powerup mechanics**: Invincibility, invisibility, infravision, extravision duration tracking
- **Oxygen/vacuum handling**: Depletion in vacuum, replenishment in air, suffocation deaths
- **Death management**: Kill player, handle reincarnation delays, manage dead player state
- **Serialization**: Pack/unpack player data for save games and network transmission
- **XML configuration**: Support MML-based configuration of initial items, damage responses, powerups, shapes

## External Dependencies
- **Map/World**: `map.h` (polygons, collision, damage polygons), `world.h` (3D points, vectors)
- **Monsters**: `monster_definitions.h`, `monsters.h` (used to treat player as monster for physics; guided missile activation)
- **Items/Weapons**: `items.h`, `weapons.h`, `projectiles.h` (player inventory, weapon state, initial items)
- **Network**: `network.h`, `network_games.h` (netdead detection, network parameters, team systems)
- **Audio**: `SoundManager.h` (damage/powerup/breathing sounds)
- **Input/Physics**: `ActionQueues.h`, `ChaseCam.h` (action queue system, camera)
- **Effects**: `fades.h` (damage fade effects), `effects.h` (impact effects referenced in damage definitions)
- **UI**: `interface.h`, `game_window.h`, `computer_interface.h` (terminal mode, screen output)
- **Serialization**: `Packing.h` (pack/unpack helper macros)
- **Scripting**: `lua_script.h` (Lua integration)

# Source_Files/GameWorld/player.h
## File Purpose

Defines the player data structure, physics model, action flags, and game settings for player management in the Aleph One engine. Provides interfaces for player initialization, update, damage handling, and physics calculations including support for networked multiplayer games.

## Core Responsibilities

- Define and manage player state (`player_data` structure) including location, health, weapons, and status
- Configure player physics models and motion (gravity, acceleration, elevation)
- Encode input actions via flag bits (movement, looking, weapon cycling, triggers)
- Track damage given/taken and maintain player statistics
- Provide player settings (energy, oxygen, powerup durations) configurable via XML
- Support teleporting, interlevel transport, and inventory management
- Interface with physics engine for position/velocity updates and collision
- Support action queue buffering for network synchronization (0x400 buffer diameter)

## External Dependencies

- **`cseries.h`:** Common utility types and macros (fixed-point, basic data structures).
- **`world.h`:** World coordinate system (angle, world_distance, point/vector types, trigonometry).
- **`map.h`:** Map structure constants (TICKS_PER_MINUTE, TELEPORTING_DURATION, damage types, game options).
- **`XML_ElementParser.h`:** XML parsing interface for MML configuration.
- **`weapons.h`:** Weapon data structures and constants (for `player_weapon_data`, trigger states).
- **Defined elsewhere:** `ActionQueues` class (action queue management for network sync); monster and object systems (for damage tracking and rendering).

**Macros for flag access** (e.g., `PLAYER_IS_DEAD()`, `SET_PLAYER_ZOMBIE_STATUS()`) provide bit-level manipulation of player state flags defined in `player_data.flags`.

# Source_Files/GameWorld/projectile_definitions.h
## File Purpose

Defines the static properties and behavior flags for all projectile types in the game engine. Contains a comprehensive data-driven lookup table of ~40 projectile definitions initialized at engine startup, each specifying damage, visual effects, physics behavior, and behavioral flags.

## Core Responsibilities

- Define projectile behavior flags (e.g., guided, gravity-affected, penetrating, melee-capable)
- Declare the `projectile_definition` struct containing all per-projectile properties
- Initialize `original_projectile_definitions[]` constant array with full definitions for all ~40 projectile types (player weapons, alien projectiles, special effects)
- Maintain a mutable `projectile_definitions[]` array for runtime modifications
- Provide serialization/deserialization functions for save/load and network transmission

## External Dependencies

- **Types referenced (defined elsewhere):** `world_distance`, `_fixed`, `damage_definition`, `int16`, `uint32`, `uint8`, `size_t`
- **Macros:** `WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_THREE_FOURTHS`, `WORLD_ONE/N` (distance scaling), `BUILD_COLLECTION()`, `NONE`
- **External symbols (enum values):** Damage types (`_damage_explosion`, `_damage_projectile`, `_damage_flame`, etc.), effect IDs (`_effect_rocket_explosion`, `_effect_grenade_explosion`, etc.), collection IDs (`_collection_rocket`, `_collection_fighter`, `_collection_compiler`, etc.), sound IDs (`_snd_rocket_flyby`, `_snd_fusion_flyby`, etc.), flags (`_alien_damage`)
- **Guard:** `#ifndef DONT_REPEAT_DEFINITIONS` allows conditional inclusion of the definitions array.

# Source_Files/GameWorld/projectiles.cpp
## File Purpose
Implements projectile physics simulation, collision detection, and lifecycle management for the Marathon/Aleph One game engine. Handles creation, movement, impact detection, and destruction of projectiles with support for gravity, bouncing, media penetration, guided targeting, and damage-on-impact.

## Core Responsibilities
- Projectile instantiation and initialization with velocity/elevation
- Per-tick physics simulation (gravity, wander, guided steering)
- Collision detection against terrain, monsters, scenery, and fluid surfaces
- Media boundary penetration handling (PMB flag special case)
- Damage application (direct hit or area-of-effect)
- Detonation effects and contrail generation
- Resource loading and serialization (packing/unpacking)
- Projectile cleanup on removal or monster owner death (orphaning)

## External Dependencies
- **Notable includes:** cseries.h (memory, macros), map.h (polygon/object access), effects.h (effect spawning), monsters.h (monster damage), player.h (player indexing), scenery.h (scenery damage), media.h (media data), SoundManager.h (audio), items.h (item management), projectile_definitions.h (definition array), dynamic_limits.h (max object counts), Packing.h (serialization), lua_script.h (Lua callbacks)
- **Defined elsewhere:** `projectiles`, `projectile_definitions`, all polygon/object/monster data accessors, `dynamic_world`, `current_player_index`, `temporary` (debug buffer), global_random(), sine_table/cosine_table, various distance/intersection math functions

# Source_Files/GameWorld/projectiles.h
## File Purpose
Header file for the projectile subsystem in the Marathon/Aleph One game engine. Defines the projectile data structure, enumeration of 39 projectile types (weapons, effects, enemy attacks), and provides functions for creating, updating, colliding, and managing active projectiles in the game world.

## Core Responsibilities
- Define enumeration of projectile types (rockets, grenades, bullets, energy bolts, fusion dispersals, drain effects, etc.)
- Maintain and expose the global `ProjectileList` vector of active projectiles
- Provide CRUD operations for projectiles: creation, movement, removal, and cleanup
- Handle projectile collision detection and detonation via `translate_projectile()`
- Manage projectile owner relationships (track owner index/type, handle orphaning when owner dies)
- Provide serialization (pack/unpack) for projectile data and definitions
- Query projectile attributes (e.g., `ProjectileIsGuided()`)
- Support contrail/trail effects and flyby sound cues

## External Dependencies
- **dynamic_limits.h**: Provides `get_dynamic_limit()` to fetch runtime limit on max projectiles
- **world.h**: Defines `angle`, `world_distance`, `world_point3d`, `world_vector3d`, trigonometry & distance functions
- Implicit: engine defines `_fixed` (fixed-point numeric type), `NONE` constant, and polygon/monster/object indexing conventions

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

## External Dependencies
- `cseries.h` ΓÇö core types, macros, utilities
- `<vector>` ΓÇö STL vector for dynamic scenery tracking
- `ShapesParser.h` ΓÇö shape descriptor parsing
- `map.h` ΓÇö object_data, new_map_object(), object owner/solidity macros, MAXIMUM_OBJECTS_PER_MAP
- `effects.h` ΓÇö new_effect()
- `scenery.h`, `scenery_definitions.h` ΓÇö scenery definitions array and types
- XML parser infrastructure (XML_ElementParser base class, Shape_GetParser(), Shape_SetPointer())
- Undefined in this file: `animate_object()`, `randomize_object_sequence()`, `GetMemberWithBounds()`, `new_map_object()` (defined elsewhere)

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

## External Dependencies
- `#include "XML_ElementParser.h"` ΓÇô XML parsing framework
- `object_location` ΓÇô defined elsewhere (game world location struct)
- `world_distance` ΓÇô defined elsewhere (coordinate/distance type)
- Comment notes Lua integration support (`ghs: allow Lua to add and delete scenery`)

# Source_Files/GameWorld/scenery_definitions.h
## File Purpose
Defines static data for environmental scenery objects in the game world. Provides flag constants, a scenery struct template, and a compiled array of 61 pre-configured scenery instances grouped by environment theme (lava, water, sewage, alien, Jjaro).

## Core Responsibilities
- Define scenery property flags (_scenery_is_solid, _scenery_is_animated, _scenery_can_be_destroyed)
- Provide the `scenery_definition` struct for static scenery configuration
- Supply a static array of 61 environment-specific scenery definitions
- Encode collision properties (radius, height) for each scenery object
- Link destroyed scenery to visual effects and replacement shapes

## External Dependencies
- **Macros**: `BUILD_DESCRIPTOR(collection, index)` ΓÇö constructs shape references; defined elsewhere
- **Collections**: `_collection_scenery1` through `_collection_scenery5` ΓÇö shape/sprite collections referenced by indices
- **Effect constants**: `_effect_lava_lamp_breaking`, `_effect_water_lamp_breaking`, `_effect_sewage_lamp_breaking`, `_effect_alien_lamp_breaking`, `_effect_grenade_explosion`, `NONE` ΓÇö define what happens when scenery is destroyed
- **Scale constants**: `WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_ONE_FOURTH` ΓÇö unit measurements for collision radius and height

# Source_Files/GameWorld/TickBasedCircularQueue.h
## File Purpose
Defines a family of tick-indexed circular queue data structures for storing per-frame game state (typically action flags). Supports concurrent reader/writer access patterns where the reader consumes past ticks and the writer enqueues new ticks, with careful thread-safety properties enabling lock-free usage in specific scenarios.

## Core Responsibilities
- Manage a circular buffer indexed by game tick (frame number) with separate read/write cursors
- Provide concurrent reader/writer interface with documented thread-safety invariants
- Support peeking at arbitrary ticks and consuming from the read end
- Track available capacity and queue size
- Broadcast writes to multiple child queues (DuplicatingTickBasedCircularQueue variant)
- Allow in-place mutation of enqueued elements before consumption (MutableElementsTickBasedCircularQueue variant)

## External Dependencies
- `cseries.h` ΓÇô provides type definitions (`int32`, etc.) and compatibility layer
- `<set>` ΓÇô STL container used by DuplicatingTickBasedCircularQueue to store child queue pointers

# Source_Files/GameWorld/weapon_definitions.h
## File Purpose
Defines weapon system data structures and configuration for the Aleph One game engine (Marathon). Specifies weapon mechanics, animations, ammunition, audio, physics (recoil, projectiles), and shell casing behavior. Contains hardcoded definitions for 10 weapons and serialization helpers.

## Core Responsibilities
- Define weapon classes (melee, single-trigger, dual-function, dual-wield, multipurpose)
- Define weapon behavior flags (automatic, overload, phase-firing, underwater-capable, ammo-sharing, etc.)
- Configure weapon animations (idle, firing, reloading shapes and timing)
- Specify trigger mechanics per weapon (ammo type, fire rate, charging, burst count, recoil)
- Define shell casing physics and rendering (initial position, velocity, acceleration)
- Store weapon ordering/slot assignments
- Provide pack/unpack serialization for weapon data

## External Dependencies
- **Types (defined elsewhere):** `_fixed` (fixed-point number), `world_distance` (coordinate type), `uint8`
- **Constants:** `FIXED_ONE`, `FIXED_ONE_HALF`, `TICKS_PER_SECOND`, `WORLD_ONE_FOURTH`, `NONE`
- **External IDs referenced:** weapon item types (`_i_*`), projectile types (`_projectile_*`), sound IDs (`_snd_*`), collection IDs (`_collection_*`, `_weapon_in_hand_collection`)
- **Defines:** `NUMBER_OF_TRIGGERS` (inferred as 2 from trigger array usage), `NUMBER_OF_SHELL_CASING_TYPES` (5), `NUMBER_OF_WEAPONS` (10)

# Source_Files/GameWorld/weapons.cpp
## File Purpose
Core weapon system implementation for the Aleph One engine (Marathon). Manages player weapon state machines, firing/reloading cycles, ammunition tracking, visual animations (including shell casings), and weapon-to-environment compatibility. Handles both per-trigger dual-wield mechanics and weapon switching.

## Core Responsibilities
- Weapon state machine updates (idle, firing, reloading, recovering, charging, etc.)
- Trigger (primary/secondary) management and action polling
- Ammunition tracking, loading, and depletion
- Weapon raising/lowering animations and sequences
- Shell casing visual effect spawning and updating
- Fired projectile creation and ballistic calculations
- Weapon readiness and environment compatibility checks
- Player item pickup processing for ammo and weapon acquisition
- Game state serialization/deserialization (packing/unpacking)
- XML configuration parsing for weapon definitions and shell casings

## External Dependencies
- **Core engine:** `map.h` (world geometry), `projectiles.h` (fire projectile), `player.h` (player state), `interface.h` (collections, shapes), `items.h` (item types), `monsters.h` (monster references), `game_window.h` (UI updates)
- **Audio:** `SoundManager.h` (play weapon/shell casing sounds)
- **Serialization:** `Packing.h` (stream macros)
- **Configuration:** `weapon_definitions.h` (weapon/trigger/shell casing definitions, loaded from external config/XML), `XML_ElementParser.h` (XML parsing base class)
- **Standard library:** `<string.h>`, `<stdlib.h>`, `<limits.h>` (for SHRT_MAX, SHRT_MIN)
- **Platform:** `#pragma segment weapons` for 68k Macintosh (legacy)

**Defined elsewhere (referenced but not defined here):**
- `weapon_definitions`, `shell_casing_definitions`, `weapon_ordering_array` (external definition arrays)
- `dynamic_world`, `current_player_index` (global game state)
- `get_dynamic_limit()`, `get_player_data()`, `get_item_kind()` (engine accessors)
- `GetMemberWithBounds()` (bounds-checked array access macro)

# Source_Files/GameWorld/weapons.h
## File Purpose
Header file defining the weapon system for a game engine (Aleph One). Declares data structures for weapon state per player and per trigger, enumerations for weapon types and actions, and function prototypes for weapon initialization, updates, serialization, and queries.

## Core Responsibilities
- Define weapon type identifiers and weapon action states (idle, charging, firing)
- Model weapon state: per-player inventory, per-weapon trigger state (ammo, fire rate), and shell casing physics
- Manage weapon display/rendering information (positioning, animation frames, visual modes)
- Initialize and update weapon systems at game startup, new game, and per-frame
- Handle ammunition tracking and reloading when items are picked up
- Support serialization (pack/unpack) for save/load and network sync
- Provide queries for UI/display (current weapon, ammo, firing state)
- Integrate XML-based weapon configuration

## External Dependencies
- **Custom typedefs**: `_fixed` (fixed-point), `uint16`, `uint8`, `int16`, `int32`, `size_t`, `bool`
- **XML support**: References `XML_ElementParser` (defined elsewhere; used for weapon configuration parsing)
- **Legacy API**: `get_weapon_array()`, `calculate_weapon_array_length()` marked as superseded by pack/unpack routines
- **No direct #include directives shown** ΓÇö definitions of `_fixed`, `uint*`, etc. provided by project header chain
- Comments reference `player.c`, `lua_script.cpp` (consumers of these structures)

# Source_Files/GameWorld/world.cpp
## File Purpose

Core geometric and mathematical engine for Aleph One's game world. Implements trigonometric lookups, 2D/3D point transformations, distance calculations, and random number generation. Includes specialized long-distance coordinate overflow handling for large map coordinates.

## Core Responsibilities

- Initialize and provide access to precomputed sine/cosine/tangent lookup tables
- Translate and rotate 2D/3D points in world space around origins
- Calculate arctangent from Cartesian coordinates using binary search
- Compute Euclidean and approximate distances between points
- Provide deterministic random number generation with global and local seeds
- Handle integer square root and overflow-extended coordinate arithmetic

## External Dependencies

- **Includes:** `<stdlib.h>`, `<math.h>`, `<limits.h>`, `cseries.h` (base types/macros), `world.h` (declarations)
- **Defined elsewhere:** `cosine_table`, `sine_table` (extern globals, initialized here); `TRIG_MAGNITUDE`, `NUMBER_OF_ANGLES`, `NORMALIZE_ANGLE()` macro (world.h); `int16`, `int32`, `uint16` (cstypes.h); `GUESS_HYPOTENUSE()`, `INT16_MAX`, `INT32_MIN` macros

# Source_Files/GameWorld/world.h
## File Purpose
Header defining the fundamental coordinate system, geometric types, and transformation utilities for the game world. Establishes fixed-point arithmetic conventions, angle representation (512 angles per circle), and 2D/3D point/vector structures used throughout the engine.

## Core Responsibilities
- Define fixed-point world coordinate types (`world_distance`, `angle`) and their conversion macros
- Declare geometric structures: world points/vectors (2D, 3D), fixed-point variants, and long-integer overflow variants
- Provide angle normalization and discretized facing-direction macros (4/5/8-way)
- Declare geometric transformation functions (rotate, translate, transform) for 2D and 3D
- Declare trigonometric table access and angle calculation (`arctangent`)
- Provide distance/hypotenuse calculation utilities
- Declare random number generation and seed management
- Declare overflow-safe 2D transformation variants for long-distance calculations

## External Dependencies
- `_fixed`, `_fixed` arithmetic (defined elsewhere; used for fixed-point types)
- `isqrt()` ΓÇö integer square root (likely in `world.c`)
- Trigonometric tables declared but populated by `build_trig_tables()` in `world.c`
- `FIXED_FRACTIONAL_BITS` macro (defined elsewhere, used in coordinate conversions)


