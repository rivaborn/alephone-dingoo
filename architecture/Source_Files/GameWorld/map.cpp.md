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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `static_data` | struct | Static level data (environment code, music, mission flags, level name) |
| `dynamic_data` | struct | Dynamic per-frame state (tick count, entity counts, random seed, game info) |
| `object_data` | struct | 32-byte game object (location, shape, animation, transfer mode, owner) |
| `polygon_data` | struct | 128-byte map polygon (height, texture, sound sources, object list head) |
| `line_data` | struct | 32-byte map line segment (endpoints, adjacent polygons, solidity flags) |
| `side_data` | struct | 64-byte wall side (textures, transfer modes, control panels, lightsources) |
| `endpoint_data` | struct | 16-byte map vertex (flags, elevation, floor/ceiling heights) |
| `ambient_sound_image_data` | struct | Non-directional ambient sound source |
| `random_sound_image_data` | struct | Directional random sound effect with period/pitch variation |
| `map_object` | struct | Initial object placement (type, index, location, polygon, flags) |
| `shape_and_transfer_mode` | struct | Object rendering state (collection, shape index, animation frame/phase) |
| `XML_TextureEnvironmentParser` | class | XML parser for texture-environment slot configuration |
| `XML_TextureLoadingParser` | class | XML parser for landscape texture loading toggle |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `static_world` | `static_data*` | global | Current level's static data |
| `dynamic_world` | `dynamic_data*` | global | Current level's dynamic frame state |
| `ObjectList` | `vector<object_data>` | global | All objects in current map |
| `MonsterList` | `vector<monster_data>` | global | All monsters in current map |
| `ProjectileList` | `vector<projectile_data>` | global | All projectiles in current map |
| `EffectList` | `vector<effect_data>` | global | All visual effects in current map |
| `PolygonList` | `vector<polygon_data>` | global | All map polygons |
| `LineList` | `vector<line_data>` | global | All map lines |
| `SideList` | `vector<side_data>` | global | All map sides |
| `EndpointList` | `vector<endpoint_data>` | global | All map vertices |
| `PlatformList` | `vector<platform_data>` | global | All moving platforms |
| `AmbientSoundImageList` | `vector<ambient_sound_image_data>` | global | Ambient sound sources |
| `RandomSoundImageList` | `vector<random_sound_image_data>` | global | Random sound effects |
| `MapIndexList` | `vector<int16>` | global | Sound source index buffer |
| `AutomapLineList` | `vector<uint8>` | global | Bitfield: lines visible on automap |
| `AutomapPolygonList` | `vector<uint8>` | global | Bitfield: polygons visible on automap |
| `MapAnnotationList` | `vector<map_annotation>` | global | Annotation text overlays |
| `SavedObjectList` | `vector<map_object>` | global | Initial object placements from map file |
| `Environments` | `short[5][7]` | static | EnvironmentΓåÆcollection mappings (Lh'owon water/lava/sewage, Jjaro, Pfhor) |
| `IntersectedObjects` | `vector<short>` | static | Reusable buffer for collision checks |
| `LandscapesLoaded` | `bool` | global | Whether M2/Infinity landscape textures loaded (M1 compat toggle) |
| `LoadedWallTexture` | `short` | global | First wall texture collection loaded (for infravision fog) |
| `OriginalEnvironments` | `short**` | static | Backup of `Environments` for XML reset |
| `map_collections` | `bool[NUMBER_OF_COLLECTIONS]` | static | Tracks which collections visible in current level |
| `media_effects` | `bool[NUMBER_OF_EFFECT_TYPES]` | static | Tracks which media detonation effects visible |
| `game_is_networked` | `bool` | global | True if network multiplayer game |

## Key Functions / Methods

### allocate_map_memory
- **Signature:** `void allocate_map_memory(void)`
- **Purpose:** Allocate and clear main world structures and all entity lists for a map load.
- **Inputs:** None (uses global state).
- **Outputs/Return:** None; modifies global `static_world`, `dynamic_world`, and all vector lists.
- **Side effects:** Allocates heap memory; calls `allocate_player_memory()`.
- **Calls:** `new`, `obj_clear()`, `allocate_player_memory()`.
- **Notes:** Must be called before any map data is loaded.

### initialize_map_for_new_game
- **Signature:** `void initialize_map_for_new_game(void)`
- **Purpose:** Clear dynamic world for a new game, reinitialize players and monsters.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Resets `dynamic_world`; calls `initialize_players()`, `initialize_monsters()`.
- **Calls:** `obj_clear()`, `initialize_players()`, `initialize_monsters()`.

### initialize_map_for_new_level
- **Signature:** `void initialize_map_for_new_level(void)`
- **Purpose:** Clear level state while preserving game-wide counters (player count, tick count, random seed, civilian counts).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Clears `dynamic_world` and `static_world` except preserved fields; clears all entity lists (automap lines/polygons, effects, projectiles, monsters, objects).
- **Calls:** `obj_clear()`, `Console::instance()->clear_saves()`, `objlist_clear()`.
- **Notes:** Persists `game_information`, `player_count`, `tick_count`, `random_seed`, civilian counts, and `speaking_player_index`.

### mark_map_collections
- **Signature:** `void mark_map_collections(bool loading)`
- **Purpose:** Scan map geometry and initial objects to determine which texture collections are needed, then mark them for load/unload.
- **Inputs:** `loading` ΓÇö true to load, false to unload.
- **Outputs/Return:** None.
- **Side effects:** Updates `map_collections[]` and `media_effects[]` flags; calls `mark_collection_for_loading/unloading()` and `mark_effect_collections()`.
- **Calls:** `mark_collection_for_loading()`, `mark_collection_for_unloading()`, `mark_effect_collections()`, `get_media_data()`, `get_media_collection()`, `get_media_detonation_effect()`, `get_scenery_collection()`, `get_damaged_scenery_collection()`.
- **Notes:** Scans all polygons, sides, media, and initial objects (saved_objects).

### reconnect_map_object_list
- **Signature:** `void reconnect_map_object_list(void)`
- **Purpose:** Rebuild the linked-list pointers linking objects to their containing polygons.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Updates `polygon->first_object` and `object->next_object` for all objects.
- **Calls:** `get_polygon_data()`, `get_object_data()`, `SLOT_IS_USED()`.
- **Notes:** Used when loading a saved game or after network prediction rollback.

### new_map_object, new_map_object2d, new_map_object3d
- **Signature:**
  - `short new_map_object(object_location *location, shape_descriptor shape)`
  - `short new_map_object2d(world_point2d *location, short polygon_index, shape_descriptor shape, angle facing)`
  - `short new_map_object3d(world_point3d *location, short polygon_index, shape_descriptor shape, angle facing)`
- **Purpose:** Create a new object in the world at a given location, insert into polygon's object list, and return object index.
- **Inputs:** Location (2D/3D or `object_location` struct with flags), polygon index, shape descriptor, facing angle.
- **Outputs/Return:** Object index, or NONE if no free slot.
- **Side effects:** Allocates an object slot, links it to polygon, sets shape and invisibility flags.
- **Calls:** `_new_map_object()`, `get_polygon_data()`, `get_object_data()`, `SET_OBJECT_INVISIBILITY()`.
- **Notes:** Hanging-from-ceiling flag affects Z-coordinate calculation; `new_map_object()` wraps 2D version.

### valid_point2d, valid_point3d
- **Signature:**
  - `bool valid_point2d(world_point2d *p)`
  - `bool valid_point3d(world_point3d *p)`
- **Purpose:** Check whether a point is within valid map geometry (inside a polygon and, for 3D, between floor/ceiling).
- **Inputs:** Point location.
- **Outputs/Return:** True if valid, false otherwise.
- **Side effects:** None.
- **Calls:** `world_point_to_polygon_index()`, `get_polygon_data()`.

### line_is_obstructed
- **Signature:** `bool line_is_obstructed(short polygon_index1, world_point2d *p1, short polygon_index2, world_point2d *p2)`
- **Purpose:** Trace a line segment across polygon boundaries to determine if a solid line or media boundary blocks it.
- **Inputs:** Starting polygon and point, ending polygon and point.
- **Outputs/Return:** True if obstructed (solid line or mismatched destination polygon).
- **Side effects:** None.
- **Calls:** `_find_line_crossed_leaving_polygon()`, `find_adjacent_polygon()`, `get_line_data()`, `LINE_IS_SOLID()`.
- **Notes:** Used for line-of-sight checks; crosses media boundaries transparently but reports obstruction if endpoints are in different media.

### point_is_player_visible, point_is_monster_visible
- **Signature:**
  - `bool point_is_player_visible(short max_players, short polygon_index, world_point2d *p, int32 *distance)`
  - `bool point_is_monster_visible(short polygon_index, world_point2d *p, int32 *distance)`
- **Purpose:** Check if a point is visible from any active player/monster and return minimum distance.
- **Inputs:** Point polygon, point location, max player count (for player version), out pointer for distance.
- **Outputs/Return:** True if visible from at least one observer; distance set to INT32_MAX if not visible.
- **Side effects:** Populates `IntersectedObjects` vector (for monsters version).
- **Calls:** `get_player_data()`, `get_monster_data()`, `get_object_data()`, `line_is_obstructed()`, `guess_distance2d()`, `possible_intersecting_monsters()`.

### turn_object_to_shit
- **Signature:** `void turn_object_to_shit(short garbage_object_index)`
- **Purpose:** Convert a dead object into a garbage object (corpse), enforcing per-polygon and per-map corpse limits.
- **Inputs:** Object index (usually a monster being turned to corpse).
- **Outputs/Return:** None.
- **Side effects:** Sets object owner to `_object_is_garbage`, possibly removes another garbage object to respect limits.
- **Calls:** `get_object_data()`, `get_polygon_data()`, `GET_OBJECT_OWNER()`, `SET_OBJECT_OWNER()`, `remove_map_object()`.
- **Notes:** Respects `graphics_preferences->double_corpse_limit`.

### animate_object
- **Signature:** `void animate_object(short object_index)`
- **Purpose:** Advance object animation frame/phase by one tick.
- **Inputs:** Object index.
- **Outputs/Return:** None.
- **Side effects:** Updates `object->sequence` frame and phase; sets animation flags; may call sound functions.
- **Calls:** `get_object_data()`, `get_object_shape_and_transfer_mode()`, frame check logic.
- **Notes:** Called once per tick per object; logs warnings for infinite loops in animation.

### play_object_sound, play_polygon_sound, play_side_sound, play_world_sound
- **Signature:**
  - `void play_object_sound(short object_index, short sound_code)`
  - `void play_polygon_sound(short polygon_index, short sound_code)`
  - `void _play_side_sound(short side_index, short sound_code, _fixed pitch)`
  - `void play_world_sound(short polygon_index, world_point3d *origin, short sound_code)`
- **Purpose:** Queue a sound at the specified location.
- **Inputs:** Object/polygon/side/world index or location, sound code, optional pitch.
- **Outputs/Return:** None.
- **Side effects:** Calls `SoundManager::instance()->PlaySound()` with location and owner info.
- **Calls:** `get_object_data()`, `get_monster_data()`, `get_polygon_data()`, `get_side_data()`, `find_center_of_polygon()`, `SoundManager::instance()->PlaySound()`.
- **Notes:** Uses object's or monster's sound location; polygon sounds play at polygon center.

### handle_random_sound_image
- **Signature:** `void handle_random_sound_image(void)`
- **Purpose:** Update random sound phases and trigger playback for sounds in the current player's polygon.
- **Inputs:** None (uses `current_player`).
- **Outputs/Return:** None.
- **Side effects:** Plays sound via `SoundManager`; updates image phase.
- **Calls:** `get_polygon_data()`, `get_random_sound_image_data()`, `SoundManager::instance()->DirectPlaySound()`.
- **Notes:** Called once per frame; validates image pointer before use.

### _sound_listener_proc
- **Signature:** `world_location3d *_sound_listener_proc(void)`
- **Purpose:** Return the listener (camera) location for sound obstruction calculations.
- **Inputs:** None.
- **Outputs/Return:** Pointer to current player's camera location, or NULL if not in-game.
- **Side effects:** None.
- **Calls:** `get_game_state()`.

### _sound_obstructed_proc
- **Signature:** `uint16 _sound_obstructed_proc(world_location3d *source)`
- **Purpose:** Determine sound obstruction flags (solid line block, media boundary, muffling) between source and listener.
- **Inputs:** Sound source location (3D + polygon).
- **Outputs/Return:** Bitfield of obstruction flags (`_sound_was_obstructed`, `_sound_was_media_obstructed`, `_sound_was_media_muffled`).
- **Side effects:** None.
- **Calls:** `_sound_listener_proc()`, `line_is_obstructed()`, `get_polygon_data()`, `get_media_data()`.
- **Notes:** Handles submersion in media; muffles if source and listener in same media but one is submerged differently.

### _sound_add_ambient_sources_proc
- **Signature:** `void _sound_add_ambient_sources_proc(void *data, add_ambient_sound_source_proc_ptr add_one_ambient_sound_source)`
- **Purpose:** Enumerate all ambient sound sources in the listener's polygon and nearby, queuing them for playback with proper flags.
- **Inputs:** Ambient sound data pointer, callback function to add sound.
- **Outputs/Return:** None.
- **Side effects:** Calls callback multiple times with ambient sources.
- **Calls:** `_sound_listener_proc()`, `get_polygon_data()`, `get_platform_data()`, `get_media_data()`, `get_map_indexes()`, `get_light_intensity()`, multiple callback invocations.
- **Notes:** Handles media ambient sounds, platform moving sounds, and sound sources from saved_objects list.

### change_polygon_height
- **Signature:** `bool change_polygon_height(short polygon_index, world_distance new_floor_height, world_distance new_ceiling_height, struct damage_definition *damage)`
- **Purpose:** Change polygon floor/ceiling height, check for height collision, damage entities inside, and notify platforms.
- **Inputs:** Polygon index, new heights, optional damage definition.
- **Outputs/Return:** True if height change is legal (no object collision); false otherwise.
- **Side effects:** May adjust objects/monsters, trigger damage, activate platforms/lights.
- **Calls:** `get_polygon_data()`, `adjusted_height_check()`, `cause_polygon_damage()`, `adjust_monster_for_polygon_height_change()`, etc.
- **Notes:** Preserves object/monster state if heights are invalid (legal_change=false).

### line_has_variable_height
- **Signature:** `bool line_has_variable_height(short line_index)`
- **Purpose:** Check if a line's height varies (has a platform on one or both adjacent polygons).
- **Inputs:** Line index.
- **Outputs/Return:** True if either adjacent polygon is a platform.
- **Side effects:** None.
- **Calls:** `get_line_data()`, `get_polygon_data()`.

### random_point_on_circle
- **Signature:** `void random_point_on_circle(world_point3d *center, short center_polygon_index, world_distance radius, world_point3d *random_point, short *random_polygon_index)`
- **Purpose:** Generate a random point on a circle at the center's Z-height, keeping it in valid geometry.
- **Inputs:** Circle center (3D), center polygon, radius.
- **Outputs/Return:** Random point and its polygon index (or NONE if invalid).
- **Side effects:** None.
- **Calls:** `translate_point2d()`, `keep_line_segment_out_of_walls()`, `find_new_object_polygon()`, `get_polygon_data()`, `global_random()`.

### _find_line_crossed_leaving_polygon (private)
- **Signature:** `short _find_line_crossed_leaving_polygon(short polygon_index, world_point2d *p0, world_point2d *p1, bool *last_line)`
- **Purpose:** Determine which line of a polygon is crossed when moving from p0 to p1; used by `line_is_obstructed()` to trace rays.
- **Inputs:** Polygon index, two 2D points, pointer to flag.
- **Outputs/Return:** Line index crossed, or NONE if destination is in polygon; `last_line` set if p1 is exactly on a line.
- **Side effects:** None.
- **Calls:** `get_polygon_data()`, `get_endpoint_data()`.
- **Notes:** Uses cross-product tests for geometric intersection; called repeatedly in ray-tracing loop.

### _new_map_object (private)
- **Signature:** `static short _new_map_object(shape_descriptor shape, angle facing)`
- **Purpose:** Find a free object slot, initialize it with shape/facing/defaults, and mark slot in-use.
- **Inputs:** Shape descriptor, facing angle.
- **Outputs/Return:** Object index, or NONE if no free slots.
- **Side effects:** Allocates and initializes object slot; sets shape, zeroes animation/transfer mode, sets visibility for UNONE shape.
- **Calls:** `SLOT_IS_FREE()`, `MARK_SLOT_AS_USED()`, `SET_OBJECT_INVISIBILITY()`.
- **Notes:** Core utility called by all object creation functions.

### mark_environment_collections, collection_in_environment
- **Signature:**
  - `void mark_environment_collections(short environment_code, bool loading)`
  - `bool collection_in_environment(short collection_code, short environment_code)`
- **Purpose:** Load/unload all texture collections for an environment; check membership.
- **Inputs:** Environment code (0ΓÇô4: Lh'owon water/lava/sewage, Jjaro, Pfhor), loading flag (or collection code for membership check).
- **Outputs/Return:** None for mark; boolean for membership check.
- **Side effects:** Calls `mark_collection_for_loading/unloading()` and landscape collection markers.
- **Calls:** `mark_collection_for_loading()`, `mark_collection_for_unloading()`.
- **Notes:** Sets `LoadedWallTexture` to first loaded collection for infravision support; respects `LandscapesLoaded` flag.

### XML_TextureEnvironmentParser
- **Signature:** `class XML_TextureEnvironmentParser : public XML_ElementParser`
- **Purpose:** Parse XML elements specifying texture collection assignments to environment slots.
- **Key methods:**
  - `Start()`: Back up original `Environments` array.
  - `HandleAttribute()`: Parse `index`, `which`, `coll` attributes.
  - `AttributesDone()`: Validate all attributes present, update `Environments[Index][Which] = Coll`.
  - `ResetValues()`: Restore from backup.
- **Inputs/Outputs:** XML parser callbacks (standard).
- **Notes:** Supports hot-swapping texture environments via XML patches.

## Control Flow Notes
**Map initialization pipeline:**
1. `allocate_map_memory()` ΓÇö allocate all structures
2. `initialize_map_for_new_game()` ΓÇö clear dynamic state for new game
3. `initialize_map_for_new_level()` ΓÇö preserve game counters, clear level state
4. `mark_map_collections()` ΓÇö determine needed textures
5. `mark_environment_collections()` ΓÇö load environment-specific textures

**Per-frame updates:**
- Object animation (`animate_object()`)
- Sound playback (`play_*_sound()`, `handle_random_sound_image()`)
- Visibility/collision checks (called from monster/projectile update code)

**Object lifecycle:**
- Creation: `new_map_object()` ΓåÆ `_new_map_object()` ΓåÆ inserted into polygon linked list
- Update: animation, sound, collision callbacks
- Removal: `remove_map_object()` ΓåÆ unlink from polygon, clear slot

No explicit "render" or "shutdown" phase in this file; rendering is handled by external code that calls these accessors and geometry queries.

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
