# Source_Files/RenderOther/motion_sensor.cpp

## File Purpose
Implements the motion sensor HUD displayΓÇöa circular radar that tracks nearby monsters and players. Manages entity detection, position history, rendering with fading intensity effects, and network compass indicators for team awareness in multiplayer games.

## Core Responsibilities
- **Entity lifecycle management**: Add/remove entities from tracking list with smooth fade-out on removal
- **Spatial scanning**: Periodically scan world for monsters/players within range, check line-of-sight
- **Position tracking**: Maintain 6-frame history of entity positions for distance-based intensity visualization
- **Rendering**: Draw entity blips as transparent sprites on circular sensor; handle platform-specific renderers (SW/OpenGL/Lua)
- **Network compass**: Display directional indicators for team members in multiplayer
- **XML configuration**: Allow runtime customization of sensor parameters and monsterΓåÆdisplay-type mappings

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `motion_sensor_definition` | struct | Frequency, range, and scale parameters for sensor behavior |
| `entity_data` | struct | Tracks a single entity: position history (6 frames), visibility flags, shape, linked monster index |
| `region_data` | struct | Clip bounds [x0,x1] for each scanline of circular sensor display |
| `MonsterDisplays[]` | array (static) | Maps each monster type ΓåÆ display type (Friend/Alien/Enemy) |
| `XML_MotSensAssignParser` | class | Parses `<assign>` XML tags to override monster display types |
| `XML_MotSensParser` | class | Parses `<motion_sensor>` XML for scale, range, update/rescan frequencies |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `motion_sensor_player_index` | short | file-static | Which player owns this sensor |
| `motion_sensor_side_length` | short | file-static | Width/height of circular sensor display |
| `sensor_region` | region_data* | file-static | Clipping boundaries for each row of sensor (alloc'd in init) |
| `entities` | entity_data* | file-static | Array of tracked entities (size `MAXIMUM_MOTION_SENSOR_ENTITIES`) |
| `motion_sensor_settings` | motion_sensor_definition | file-static | Sensor parameters (frequencies, range, scale) |
| `mount_shape`, `virgin_mount_shapes`, etc. | shape_descriptor | file-static | Shape references for rendering (mount, compass, entity types) |
| `network_compass_state` | short | file-static | Current compass indicators (NW/NE/SE/SW flags) |
| `ticks_since_last_update`, `ticks_since_last_rescan` | int32 | file-static | Tick counters for update/rescan intervals |
| `motion_sensor_changed` | bool | file-static | Flag: display needs redraw |
| `OriginalMonsterDisplays` | short* | file-static | Backup of default MonsterDisplays[] (for XML reset) |
| `original_motion_sensor_settings` | motion_sensor_definition* | file-static | Backup of default settings (for XML reset) |

## Key Functions / Methods

### initialize_motion_sensor
- Signature: `void initialize_motion_sensor(shape_descriptor mount, shape_descriptor virgin_mounts, shape_descriptor aliens, shape_descriptor friends, shape_descriptor enemies, shape_descriptor compasses, short side_length)`
- Purpose: Initialize motion sensor with shape assets and allocate tracking structures
- Inputs: Shape descriptors for display components; side_length = sensor diameter in pixels
- Outputs/Return: None
- Side effects: Allocates `entities[]` array, `sensor_region[]` array, stores shape references; calls `precalculate_sensor_region()`
- Calls: `new entity_data[]`, `new region_data[]`, `precalculate_sensor_region()`
- Notes: Must call `reset_motion_sensor()` after shapes are loaded

### reset_motion_sensor
- Signature: `void reset_motion_sensor(short player_index)`
- Purpose: Reset sensor state for a given player; clear entity list and restore virgin display
- Inputs: `player_index` = which player "owns" this sensor
- Outputs/Return: None
- Side effects: Clears all entity slots, copies virgin_mount bitmap to mount, resets tick counters
- Calls: `get_shape_bitmap_and_shading_table()`, `bitmap_window_copy()`, `MARK_SLOT_AS_FREE()`

### HUD_Class::motion_sensor_scan
- Signature: `void motion_sensor_scan(short ticks_elapsed)`
- Purpose: Main update entry point; scan world for entities and render display
- Inputs: `ticks_elapsed` = ticks since last call (or `NONE` to force immediate rescan)
- Outputs/Return: None
- Side effects: Updates `ticks_since_last_rescan` / `ticks_since_last_update`; modifies entity list; calls `render_motion_sensor()`
- Calls: `get_object_data()`, `get_monster_data()`, `guess_distance2d()`, `find_or_add_motion_sensor_entity()`, `render_motion_sensor()`
- Notes: If rescan timer expires, iterates all monsters within range and adds to tracking list

### render_motion_sensor (SW/OGL/Lua variants)
- Signature: `void render_motion_sensor(short ticks_elapsed)` (method of HUD_SW_Class, HUD_OGL_Class, HUD_Lua_Class)
- Purpose: Update and display motion sensor each frame
- Inputs: `ticks_elapsed`
- Outputs/Return: None
- Side effects: Erases old blips, draws current blips; sets `motion_sensor_changed` flag
- Calls: `erase_all_entity_blips()`, `draw_network_compass()`, `draw_all_entity_blips()`

### HUD_Class::erase_all_entity_blips
- Signature: `void erase_all_entity_blips(void)`
- Purpose: Shift position history, check range, update visibility, erase old blips
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies all entity position/visibility arrays; marks entities out-of-range for removal; calls `erase_entity_blip()`
- Calls: `get_object_data()`, `get_monster_data()`, `guess_distance2d()`, `erase_entity_blip()`, `memmove()`

### HUD_Class::draw_all_entity_blips
- Signature: `void draw_all_entity_blips(void)` (with Lua variant)
- Purpose: Render all active entity blips with intensity based on position-history index
- Inputs: None
- Outputs/Return: None
- Side effects: Draws to display; Lua variant calls `clear_entity_blips()` and `add_entity_blip()`
- Calls: `draw_entity_blip()` or `add_entity_blip()` (Lua)

### HUD_SW_Class::draw_or_erase_unclipped_shape
- Signature: `void draw_or_erase_unclipped_shape(short x, short y, shape_descriptor shape, bool draw)`
- Purpose: Draw or erase (restore from virgin mount) an unclipped sprite at compass cardinal positions
- Inputs: `x, y` screen coords; `shape`; `draw` flag
- Outputs/Return: None
- Side effects: Modifies mount bitmap
- Calls: `get_shape_bitmap_and_shading_table()`, `unclipped_solid_sprite_copy()`, `bitmap_window_copy()`

### HUD_SW_Class::erase_entity_blip
- Signature: `void erase_entity_blip(point2d *location, shape_descriptor shape)`
- Purpose: Erase an entity blip by copying virgin mount pixels back
- Inputs: `location` = 2D sensor coords; `shape` = for size info
- Outputs/Return: None
- Side effects: Modifies mount bitmap
- Calls: `get_shape_bitmap_and_shading_table()`, `bitmap_window_copy()`

### HUD_SW_Class::draw_entity_blip
- Signature: `void draw_entity_blip(point2d *location, shape_descriptor shape)`
- Purpose: Draw an entity blip as a transparent sprite, clipped to circular sensor boundary
- Inputs: `location` = 2D sensor coords; `shape`
- Outputs/Return: None
- Side effects: Modifies mount bitmap
- Calls: `get_shape_bitmap_and_shading_table()`, `clipped_transparent_sprite_copy()`

### static short find_or_add_motion_sensor_entity
- Signature: `static short find_or_add_motion_sensor_entity(short monster_index)`
- Purpose: Find existing entity tracking this monster, or add new one if space available
- Inputs: `monster_index`
- Outputs/Return: Entity array index (or `NONE` if no space)
- Side effects: Allocates entity slot, initializes position/visibility, sets flags
- Calls: `get_monster_data()`, `get_object_data()`, `get_motion_sensor_entity_shape()`

### static void precalculate_sensor_region
- Signature: `static void precalculate_sensor_region(short side_length)`
- Purpose: Pre-compute [x0,x1] clipping bounds for each scanline of circular sensor
- Inputs: `side_length` = diameter
- Outputs/Return: None
- Side effects: Fills `sensor_region[]` array with x-clip bounds; computes circle math
- Calls: `sqrt()`
- Notes: Assumes `sensor_region[]` already allocated

### Bitmap utility functions (static)
- **`bitmap_window_copy`**: Copy rectangular region between bitmaps (pixel-by-pixel, no transparency)
- **`clipped_transparent_sprite_copy`**: Draw sprite with per-scanline clipping and transparency (pixel 0 = transparent)
- **`unclipped_solid_sprite_copy`**: Draw sprite without clipping, all pixels copied

### static shape_descriptor get_motion_sensor_entity_shape
- Signature: `static shape_descriptor get_motion_sensor_entity_shape(short monster_index)`
- Purpose: Determine display shape (friendly/enemy/alien) based on entity type and player relationships
- Inputs: `monster_index`
- Outputs/Return: Shape descriptor for blip
- Side effects: None
- Calls: `get_monster_data()`, `get_player_data()`, `MonsterDisplays[]` lookup
- Notes: For players, checks team & game type; for monsters, uses MonsterDisplays[] table

### XML_MotSensAssignParser methods
- **`Start()`**: Backup original MonsterDisplays if not already done; reset IsPresent flags
- **`HandleAttribute()`**: Parse "monster" and "type" attributes
- **`AttributesDone()`**: Validate both attributes present, apply to MonsterDisplays[]
- **`ResetValues()`**: Restore original MonsterDisplays and free backup

### XML_MotSensParser methods
- **`Start()`**: Backup original motion_sensor_settings
- **`HandleAttribute()`**: Parse scale, range, update_frequency, rescan_frequency
- **`ResetValues()`**: Restore original settings

### MotionSensor_GetParser
- Signature: `XML_ElementParser *MotionSensor_GetParser(void)`
- Purpose: Create and return top-level XML parser for motion sensor configuration
- Inputs: None
- Outputs/Return: Pointer to MotSensParser with child MotSensAssignParser attached
- Side effects: None

## Control Flow Notes
1. **Initialization phase**: `initialize_motion_sensor()` ΓåÆ `reset_motion_sensor()`
2. **Per-frame**: `motion_sensor_scan()` called from game loop
   - Every 15 ticks: Scan world for monsters in range ΓåÆ `find_or_add_motion_sensor_entity()`
   - Every 5 ticks: Update display ΓåÆ `erase_all_entity_blips()` ΓåÆ `draw_all_entity_blips()`
3. **Entity lifecycle**: 
   - Added when first detected within range
   - Position history shifted each update
   - Marked "being removed" when out of range; fades over 6 frames
   - Fully removed when all history frames erased

## External Dependencies
- **map.h**: `world_point3d`, `world_point2d`, `world_distance`, `WORLD_ONE`, `guess_distance2d()`, `MAXIMUM_MONSTERS_PER_MAP`, `get_object_data()`, object structures
- **monsters.h**: `monster_data`, `MAXIMUM_MONSTERS_PER_MAP`, `SLOT_IS_USED()`, `MONSTER_IS_PLAYER()`, `MONSTER_IS_ACTIVE()`, `get_monster_data()`, monster type enums
- **player.h**: `player_data`, `get_player_data()`, team constants
- **network_games.h**: `get_network_compass_state()`, compass bit flags
- **interface.h**: `get_shape_bitmap_and_shading_table()`, `shape_descriptor`, `bitmap_definition`, shape loading
- **render.h**: View/rendering infrastructure
- **HUDRenderer_*.h**: Platform-specific rendering class definitions (HUD_Class, HUD_SW_Class, HUD_OGL_Class, HUD_Lua_Class)
- Standard C: `math.h`, `string.h` (memset, memmove), `stdlib.h`
