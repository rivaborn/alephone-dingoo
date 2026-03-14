# Source_Files/RenderOther/OverheadMapRenderer.h

## File Purpose
Defines the base class and configuration structures for rendering overhead (automap) displays in the game engine. Provides a virtual interface for graphics-API-agnostic map rendering, allowing subclasses to implement rendering via OpenGL, software rasterization, or other backends.

## Core Responsibilities
- Declare configuration structures for map visual properties (colors, fonts, shapes, line styles)
- Define enums for polygon types, line types, and entity types used in overhead maps
- Provide `OverheadMapClass` base class with virtual methods for rendering map elements
- Support both real automaps (explicit polygon/line data) and "false" automaps (generated via pathfinding)
- Manage player entity shape representations and directional indicators on the map

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| line_definition | struct | Visual properties for map lines (color, pen sizes indexed by scale level) |
| thing_definition | struct | Visual properties for entities like monsters/items (color, shape type, radii per scale) |
| entity_definition | struct | Directional geometry for player representation (front/rear extents, rear facing angle) |
| annotation_definition | struct | Visual properties for text annotations (color, font set indexed by scale) |
| map_name_definition | struct | Visual properties for map title display (color, font, screen offset) |
| OvhdMap_CfgDataStruct | struct | Master configuration container (polygon colors, line/thing definitions, entity shape, display flags) |
| OverheadMapClass | class | Base class defining the rendering interface via virtual methods |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| saved_automap_lines | byte* | private member | Temporary storage for automap line visibility when generating false automap |
| saved_automap_polygons | byte* | private member | Temporary storage for automap polygon visibility when generating false automap |

## Key Functions / Methods

### Render
- Signature: `void Render(overhead_map_data& Control);`
- Purpose: Main public entry point; orchestrates rendering of the complete overhead map
- Inputs: `Control` ΓÇô overhead_map_data struct containing map bounds, scale, origin, and rendering mode
- Outputs/Return: None (side effects only)
- Side effects: Invokes all virtual rendering methods; may modify automap visibility
- Calls: Virtual methods (implementation in subclasses); `transform_endpoints_for_overhead_map()`, `generate_false_automap()`, `replace_real_automap()`
- Notes: Handles both real and false autpmaps; respects draw_everything flag

### draw_polygon (virtual)
- Signature: `virtual void draw_polygon(short vertex_count, short *vertices, rgb_color& color);`
- Purpose: Render a filled polygon with given vertices and color (overridden by subclass)
- Inputs: vertex count, array of vertex indices, RGB color
- Outputs/Return: None
- Side effects: Updates rendering backend state
- Calls: None in this file
- Notes: Default empty implementation; subclasses implement graphics-specific rendering

### draw_line (virtual)
- Signature: `virtual void draw_line(short *vertices, rgb_color& color, short pen_size);`
- Purpose: Render a line segment between two endpoints with specified color and width
- Inputs: endpoint indices, RGB color, pen width
- Outputs/Return: None
- Side effects: Updates rendering backend state
- Calls: None in this file
- Notes: Default empty implementation

### draw_thing (virtual)
- Signature: `virtual void draw_thing(world_point2d& center, rgb_color& color, short shape, short radius);`
- Purpose: Render an entity (monster, item, projectile, etc.) as a shape at given location
- Inputs: center position, RGB color, shape type (_circle_thing or _rectangle_thing), radius in world units
- Outputs/Return: None
- Side effects: Updates rendering backend state
- Calls: None in this file
- Notes: Shape determines whether a circle or rectangle is rendered

### draw_player (virtual)
- Signature: `virtual void draw_player(world_point2d& center, angle facing, rgb_color& color, short shrink, short front, short rear, short rear_theta);`
- Purpose: Render player indicator showing position, facing direction, and body orientation
- Inputs: center position, facing angle (0ΓÇô512), RGB color, shrink factor for scale, front extent, rear extent, rear angle indicator
- Outputs/Return: None
- Side effects: Updates rendering backend state
- Calls: None in this file
- Notes: `shrink = OVERHEAD_MAP_MAXIMUM_SCALE - scale` for inverted scale mapping

### draw_text (virtual)
- Signature: `virtual void draw_text(world_point2d& location, rgb_color& color, char *text, FontSpecifier& FontData, short justify);`
- Purpose: Render text at specified location with font and alignment
- Inputs: location, RGB color, text string (C-string), FontSpecifier object, justification (_justify_left or _justify_center)
- Outputs/Return: None
- Side effects: Updates rendering backend state
- Calls: None in this file
- Notes: Used for annotations and map name; FontSpecifier includes size, style, metrics

### set_path_drawing / draw_path / finish_path (virtual)
- Signatures: 
  - `virtual void set_path_drawing(rgb_color& color);`
  - `virtual void draw_path(short step, world_point2d& location);`
  - `virtual void finish_path();`
- Purpose: Render a path/trajectory line (e.g., player movement history or predicted path)
- Inputs (draw_path): step index (0 for first point), location of waypoint
- Outputs/Return: None
- Side effects: Updates rendering backend state
- Calls: None in this file
- Notes: Called in sequence: set_path_drawing ΓåÆ draw_path (multiple times) ΓåÆ finish_path

### GetVertex / GetFirstVertex / GetVertexStride (static, inline)
- Signatures:
  - `static world_point2d& GetVertex(short index);`
  - `static world_point2d *GetFirstVertex();`
  - `static int GetVertexStride();`
- Purpose: Accessor helpers for map endpoint data; convenience for graphics APIs (e.g., OpenGL vertex arrays)
- Inputs: vertex index
- Outputs/Return: Reference/pointer to endpoint vertex data, stride in bytes
- Side effects: None
- Calls: `get_endpoint_data()` (defined elsewhere)
- Notes: Stride is sizeof(endpoint_data); used for optimized vertex buffer uploads

## Control Flow Notes
**Rendering pipeline:** `Render()` is the orchestrator. It likely:
1. Calls `begin_overall()` to initialize the rendering context
2. For each category (polygons, lines, things), calls begin_*() ΓåÆ draw_*() (multiple times) ΓåÆ end_*()
3. Renders text annotations and map name
4. Optionally renders player paths
5. Calls `end_overall()` to finalize

**False automap generation:** When a map lacks explicit automap data, `generate_false_automap()` uses a pathfinding cost function (`false_automap_cost_proc()`) to determine which polygons/lines are reachable, then calls `replace_real_automap()` to restore the original data after rendering.

**Auxiliary functions:** `transform_endpoints_for_overhead_map()` handles coordinate transformation from world space to overhead-map display space.

## External Dependencies

- **cseries.h** ΓÇô Core types and utilities
- **world.h** ΓÇô `world_point2d`, `world_point3d`, `angle`, world coordinate macros
- **map.h** ΓÇô Map geometry (`endpoint_data`, line/polygon accessors)
- **monsters.h** ΓÇô Monster type constants (NUMBER_OF_MONSTER_TYPES)
- **overhead_map.h** ΓÇô `overhead_map_data` struct, scale/mode constants (OVERHEAD_MAP_MINIMUM_SCALE, etc.)
- **shape_descriptors.h** ΓÇô `shape_descriptor` type
- **shell.h** ΓÇô `_get_player_color()` function, RGBColor type
- **FontHandler.h** ΓÇô `FontSpecifier` class for text rendering

**External symbols used:**
- `get_endpoint_data(short index)` ΓÇô Returns endpoint_data for a given vertex (map.h)
- `get_line_data(short line_index)` ΓÇô Returns line_data struct (map.h)
- `_get_player_color(size_t color_index, RGBColor*)` ΓÇô Maps player color index to RGB (shell.h)
