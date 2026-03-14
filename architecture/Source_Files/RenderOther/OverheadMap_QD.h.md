# Source_Files/RenderOther/OverheadMap_QD.h

## File Purpose
QuickDraw-specific concrete implementation of the overhead map renderer. Subclasses `OverheadMapClass` to provide Classic MacOS Quickdraw graphics API bindings for rendering minimap/automap features. Authored by Loren Petrich (August 2000) as part of the Aleph One engine.

## Core Responsibilities
- Override virtual rendering methods to use QuickDraw API primitives
- Render map polygons (terrain/walls) with fill colors
- Draw line primitives for walls and elevation changes
- Render entities and items as circles or rectangles
- Draw player avatar with facing direction indicator
- Render text annotations and map names with font support
- Handle path visualization for player movement tracking

## Key Types / Data Structures
None. (File declares only a class; all types referenced are inherited or passed as parameters from `OverheadMapClass` and its dependencies.)

## Global / File-Static State
None.

## Key Functions / Methods

### draw_polygon
- Signature: `void draw_polygon(short vertex_count, short *vertices, rgb_color& color)`
- Purpose: Render a filled polygon on the overhead map
- Inputs: `vertex_count` (number of vertices), `vertices` (array of vertex indices into global vertex buffer), `color` (fill color)
- Outputs/Return: None (void)
- Side effects: QuickDraw drawing to screen buffer
- Calls: Assumed QuickDraw API calls (not visible in header)
- Notes: Virtual method override; vertex indices reference transformed endpoints from base class

### draw_line
- Signature: `void draw_line(short *vertices, rgb_color& color, short pen_size)`
- Purpose: Render a line segment with specified width
- Inputs: `vertices` (endpoint indices), `color`, `pen_size` (line thickness)
- Outputs/Return: None (void)
- Side effects: QuickDraw drawing to screen buffer
- Calls: Implied QuickDraw primitives
- Notes: Virtual method override

### draw_thing
- Signature: `void draw_thing(world_point2d& center, rgb_color& color, short shape, short radius)`
- Purpose: Draw an entity/item as a shape (circle or rectangle) at map coordinates
- Inputs: `center` (world position), `color`, `shape` (enum: rectangle or circle), `radius` (size scale)
- Outputs/Return: None (void)
- Side effects: QuickDraw drawing
- Calls: Implied QuickDraw shape primitives
- Notes: Virtual method override; used for aliens, items, projectiles, checkpoints

### draw_player
- Signature: `void draw_player(world_point2d& center, angle facing, rgb_color& color, short shrink, short front, short rear, short rear_theta)`
- Purpose: Render player avatar with directional indicator
- Inputs: `center` (player world position), `facing` (player angle/direction), `color`, `shrink` (inverse scale), `front`/`rear`/`rear_theta` (entity shape parameters)
- Outputs/Return: None (void)
- Side effects: QuickDraw drawing
- Calls: Implied QuickDraw primitives
- Notes: Virtual method override; shows player heading and collision geometry

### draw_text
- Signature: `void draw_text(world_point2d& location, rgb_color& color, char *text, FontSpecifier& FontData, short justify)`
- Purpose: Render text annotation or label on the map
- Inputs: `location` (world position), `color`, `text` (C string), `FontData` (font specification), `justify` (alignment enum)
- Outputs/Return: None (void)
- Side effects: QuickDraw text rendering
- Calls: Implied QuickDraw text API
- Notes: Virtual method override; `justify` is enum (`_justify_left` or `_justify_center`)

### set_path_drawing
- Signature: `void set_path_drawing(rgb_color& color)`
- Purpose: Initialize graphics state for drawing a movement path
- Inputs: `color` (path line color)
- Outputs/Return: None (void)
- Side effects: QuickDraw state setup
- Calls: Implied QuickDraw state functions
- Notes: Virtual method override; called before sequence of `draw_path()` calls

### draw_path
- Signature: `void draw_path(short step, world_point2d& location)`
- Purpose: Draw a single segment or point of the player's movement path
- Inputs: `step` (segment index; 0 for first point), `location` (world position)
- Outputs/Return: None (void)
- Side effects: QuickDraw drawing
- Calls: Implied QuickDraw line/point primitives
- Notes: Virtual method override; called repeatedly to trace a path; `step==0` indicates path start

## Control Flow Notes
This class is instantiated by the overhead map system and used during each frame's minimap/automap rendering. The base class `OverheadMapClass::Render()` calls these virtual methods in sequence: `begin_*()` / `draw_*()` / `end_*()` for polygons, lines, things, and then player/text/path rendering. QuickDraw-specific setup and teardown occur in the overridden methods.

## External Dependencies
- **Base class**: `OverheadMapClass` (from `OverheadMapRenderer.h`)
- **Types used (defined elsewhere)**: `rgb_color`, `world_point2d`, `angle`, `FontSpecifier`
- **Included header**: `OverheadMapRenderer.h` (provides base class and enum/struct definitions)
