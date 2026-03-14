# Source_Files/RenderOther/OverheadMap_SDL.h

## File Purpose
SDL-specific implementation of the overhead map renderer. This subclass specializes `OverheadMapClass` to draw map elements (polygons, lines, entities, annotations) using SDL graphics primitives. Part of the Aleph One Marathon engine's UI subsystem.

## Core Responsibilities
- Override virtual drawing methods for SDL rendering backend
- Render polygonal map regions with color
- Draw line segments (walls/edges) with configurable pen sizes
- Render map entities (monsters, items, projectiles) as shapes
- Draw player character with direction indicator
- Render text annotations and labels
- Manage path visualization state

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OverheadMap_SDL_Class` | class | SDL concrete renderer; inherits from `OverheadMapClass` |

(Uses: `rgb_color`, `world_point2d`, `angle`, `FontSpecifier` from dependencies)

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `path_pixel` | `uint32` | instance member | Cached pixel color for path drawing |
| `path_point` | `world_point2d` | instance member | Last path point position |

## Key Functions / Methods

### draw_polygon
- Signature: `void draw_polygon(short vertex_count, short *vertices, rgb_color &color)`
- Purpose: Render a filled or outlined polygon for a map region
- Inputs: vertex count, array of vertex indices, fill color
- Outputs/Return: None
- Side effects: SDL draw calls to render polygon
- Calls: SDL rendering functions (not visible in this file)
- Notes: Vertices are indices into a global endpoint array (from base class)

### draw_line
- Signature: `void draw_line(short *vertices, rgb_color &color, short pen_size)`
- Purpose: Render a line segment (wall edge) at specified thickness
- Inputs: two vertex indices, line color, pen width
- Outputs/Return: None
- Side effects: SDL draw calls
- Calls: SDL line rendering
- Notes: `vertices` is a 2-element array of endpoint indices

### draw_thing
- Signature: `void draw_thing(world_point2d &center, rgb_color &color, short shape, short radius)`
- Purpose: Render a game entity (monster, item, projectile) as a shape
- Inputs: world position, color, shape type (_rectangle_thing or _circle_thing), radius in pixels
- Outputs/Return: None
- Side effects: SDL shape rendering
- Calls: SDL shape primitives

### draw_player
- Signature: `void draw_player(world_point2d &center, angle facing, rgb_color &color, short shrink, short front, short rear, short rear_theta)`
- Purpose: Render the player character on the map with directional indicator
- Inputs: position, facing angle, color, scale shrink factor, entity geometry parameters
- Outputs/Return: None
- Side effects: SDL drawing
- Notes: `front`, `rear`, `rear_theta` define triangular player icon geometry

### draw_text
- Signature: `void draw_text(world_point2d &location, rgb_color &color, char *text, FontSpecifier& FontData, short justify)`
- Purpose: Render text annotation on the map
- Inputs: screen position, text color, string, font descriptor, justification mode
- Outputs/Return: None
- Side effects: SDL text/font rendering

### set_path_drawing
- Signature: `void set_path_drawing(rgb_color &color)`
- Purpose: Cache the color for subsequent path segment drawing
- Inputs: RGB color value
- Outputs/Return: None
- Side effects: Sets `path_pixel` member variable

### draw_path
- Signature: `void draw_path(short step, world_point2d &location)`
- Purpose: Draw one segment of the player's movement path trail
- Inputs: step index (0 = first point), world position of this path point
- Outputs/Return: None
- Side effects: Draws or updates path visualization; updates `path_point`
- Notes: Called iteratively for each waypoint in the path

## Control Flow Notes
This class is instantiated once per game session and called during the map overlay render phase. The base class's `Render()` method calls virtual methods in sequence: `begin_overall()`, `begin_polygons()` ΓåÆ `draw_polygon()` calls ΓåÆ `end_polygons()`, `begin_lines()` ΓåÆ `draw_line()` calls ΓåÆ `end_lines()`, etc. SDL implementation translates these calls to actual screen drawing.

## External Dependencies
- **OverheadMapRenderer.h** ΓÇô base class `OverheadMapClass` with virtual interface and configuration data structures
- **SDL** ΓÇô graphics library (linked at runtime, not directly visible in headers)
- Implicit: `world.h` (types: `world_point2d`, `angle`), `FontHandler.h` (`FontSpecifier`)
