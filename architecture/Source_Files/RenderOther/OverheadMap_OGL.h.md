# Source_Files/RenderOther/OverheadMap_OGL.h

## File Purpose
OpenGL-specific implementation of the overhead map renderer. Subclass of `OverheadMapClass` that overrides virtual rendering methods to draw map geometry, entities, and annotations using OpenGL. Provides caching mechanisms for efficient batched rendering of polygons and lines.

## Core Responsibilities
- Override base-class virtual methods to implement OpenGL rendering
- Manage cached polygons and lines for batched GPU submission
- Transform and render world geometry (map polygons, walls, platforms)
- Draw game entities (players, monsters, items) and their paths
- Render text annotations and map labels with font support
- Buffer and flush cached drawing commands in `DrawCached*()` methods

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OverheadMap_OGL_Class` | class | OpenGL-specific overhead map renderer; subclass of `OverheadMapClass` |

## Global / File-Static State
None.

## Key Functions / Methods

### begin_overall / end_overall
- Purpose: Lifecycle hooks for overall rendering setup/teardown
- Notes: Likely set OpenGL state (viewport, projection matrix, scissor)

### begin_polygons / draw_polygon / end_polygons
- Purpose: Polygon batch submission; accumulate into `PolygonCache` and render when `end_polygons()` flushes
- Notes: Overloaded from base class to accept `vertex_count`, vertex indices, and color

### DrawCachedPolygons
- Purpose: Internal helper to submit cached polygon indices to OpenGL
- Notes: Called during `end_polygons()` to batch-render accumulated geometry

### begin_lines / draw_line / end_lines
- Purpose: Line batch submission; cache pen-sized lines in `LineCache`
- Notes: `draw_line()` takes endpoint indices, color, and pen size

### DrawCachedLines
- Purpose: Internal helper to submit cached line indices to OpenGL
- Notes: Called during `end_lines()` to flush lines and reset cache

### draw_thing
- Signature: `void draw_thing(world_point2d& center, rgb_color& color, short shape, short radius)`
- Purpose: Render a single entity (monster, item, projectile) at a map location
- Inputs: center position, color, shape type (rectangle/circle), radius in pixels
- Notes: Called per entity during map update

### draw_player
- Signature: `void draw_player(world_point2d& center, angle facing, rgb_color& color, short shrink, short front, short rear, short rear_theta)`
- Purpose: Render player avatar with heading indicator and shrink scaling
- Inputs: center position, facing angle, color, shrink (inverse scale), polygon offsets for player shape
- Notes: Directional shape defined in configuration

### draw_text
- Signature: `void draw_text(world_point2d& location, rgb_color& color, char *text, FontSpecifier& FontData, short justify)`
- Purpose: Render text labels (annotations, map name) on the map
- Inputs: world location, color, text string, font specification, justification (0=left, 1=center)

### set_path_drawing / draw_path / finish_path
- Purpose: Accumulate and render monster/NPC paths as polylines
- Inputs: `draw_path()` takes step index (0 for first point) and location
- Notes: Path points cached in `PathPoints` vector; `finish_path()` flushes polyline to GPU

## Control Flow Notes
Fits into the base-class rendering loop:
1. Base `Render()` calls `begin_overall()`
2. Iterates polygons ΓåÆ calls `begin_polygons()`, multiple `draw_polygon()`, `end_polygons()` (flushes)
3. Iterates lines ΓåÆ `begin_lines()`, multiple `draw_line()`, `end_lines()` (flushes)
4. Iterates entities ΓåÆ `draw_thing()`, `draw_player()` per entity
5. Text/annotations ΓåÆ multiple `draw_text()` calls
6. Paths ΓåÆ `set_path_drawing()`, sequence of `draw_path()`, `finish_path()`
7. Calls `end_overall()`

Caching allows amortization: geometry accumulated in vectors, submitted in batch calls to OpenGL.

## External Dependencies
- `OverheadMapRenderer.h` ΓÇô base class `OverheadMapClass`, configuration struct `OvhdMap_CfgDataStruct`, type definitions (`rgb_color`, `world_point2d`, `FontSpecifier`)
- `config.h` ΓÇô compile-time guard `HAVE_OPENGL`
- `<vector>` ΓÇô STL container for caching indices and points
