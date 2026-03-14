# Source_Files/RenderOther/OverheadMap_QD.cpp

## File Purpose
Implementation of a Quickdraw-based overhead map renderer for the Aleph One game engine. Provides concrete drawing primitives (polygons, lines, objects, player icon, text, paths) for the 2D overhead/minimap display on Classic MacOS systems. Subclasses `OverheadMapClass` to replace platform-agnostic rendering calls with Quickdraw-specific graphics API invocations.

## Core Responsibilities
- Render filled polygons (map geometry/walls) with color and border
- Draw line segments between pairs of vertices
- Render game objects (entities) as rectangles or circles with size/color
- Draw player avatar as a directional triangle with rear corners indicating orientation
- Render text labels with left or center justification at map coordinates
- Manage path trace drawing (initialize pen, draw connected line segments)
- Abstract Quickdraw color setup and pen configuration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` | struct/typedef | Color value passed to drawing functions |
| `world_point2d` | struct | 2D coordinate for map positions |
| `angle` | typedef | Player facing direction (normalized angle) |
| `FontSpecifier` | class | Font configuration with `Use()` activation method |
| `PolyHandle` | opaque typedef | Quickdraw polygon handle for OpenPoly/KillPoly |
| `Rect` | struct | Quickdraw rectangle for bounds specification |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SetColor` | inline function | file-static | Wrapper mapping `rgb_color` reference to Quickdraw `RGBForeColor()` |

## Key Functions / Methods

### draw_polygon
- **Signature:** `void draw_polygon(short vertex_count, short *vertices, rgb_color& color)`
- **Purpose:** Render a filled polygon on the map (typically a wall or obstacle).
- **Inputs:** Vertex count; array of vertex indices into parent class vertex store; color.
- **Outputs/Return:** None (draws to Quickdraw port).
- **Side effects:** Sets pen size to 1├ù1, foreground color, fills polygon with black pattern.
- **Calls:** `GetVertex()` (from parent), `OpenPoly()`, `MoveTo()`, `LineTo()`, `ClosePoly()`, `PenSize()`, `SetColor()`, `GetQDGlobalsBlack()`, `FillPoly()`, `KillPoly()` (all Quickdraw).
- **Notes:** Closes polygon by moving to last vertex first; fills interior with solid black regardless of input colorΓÇöcolor appears to control only border/frame.

### draw_line
- **Signature:** `void draw_line(short *vertices, rgb_color& color, short pen_size)`
- **Purpose:** Draw a single line segment between two indexed vertices.
- **Inputs:** Vertex index array (2 elements); color; pen thickness in pixels.
- **Outputs/Return:** None.
- **Side effects:** Sets foreground color, pen size; updates current Quickdraw pen position.
- **Calls:** `GetVertex()`, `SetColor()`, `PenSize()`, `MoveTo()`, `LineTo()`.
- **Notes:** Direct port of indices to coordinates; no bounds checking on array.

### draw_thing
- **Signature:** `void draw_thing(world_point2d& center, rgb_color& color, short shape, short radius)`
- **Purpose:** Render a game object (entity) at the given center point.
- **Inputs:** Center coordinate; color; shape type (`_rectangle_thing` or `_circle_thing`); radius.
- **Outputs/Return:** None.
- **Side effects:** Sets color; draws filled rect or circle outline depending on shape.
- **Calls:** `SetColor()`, `SetRect()`, `PaintRect()`, `PenSize()`, `FrameOval()`.
- **Notes:** Asymmetric bounding rect (raddown/radup split) to adjust display size; comment indicates 50% size correction from prior implementation. Rectangles are filled; circles are outlined.

### draw_player
- **Signature:** `void draw_player(world_point2d& center, angle facing, rgb_color& color, short shrink, short front, short rear, short rear_theta)`
- **Purpose:** Render the player's overhead avatar as a directional triangle showing position and facing angle.
- **Inputs:** Center position; facing angle; color; bit-shift shrink factor; front/rear distances; rear angle offset.
- **Outputs/Return:** None.
- **Side effects:** Creates and fills a 3-vertex polygon (triangle); sets pen size to 1├ù1; additionally frames the polygon perimeter.
- **Calls:** `translate_point2d()`, `normalize_angle()`, `OpenPoly()`, `MoveTo()`, `LineTo()`, `ClosePoly()`, `PenSize()`, `GetQDGlobalsBlack()`, `FillPoly()`, `FramePoly()`, `KillPoly()`.
- **Notes:** Triangle vertices: apex at front, two rear corners. Rear corners offset by ┬▒rear_theta around rear direction. Added `FramePoly()` call (perimeter) to ensure small player icons remain visible; front/rear distances are bit-shifted (divided) by shrink factor.

### draw_text
- **Signature:** `void draw_text(world_point2d& location, rgb_color& color, char *text, FontSpecifier& FontData, short justify)`
- **Purpose:** Render a text label on the overhead map with optional center or left alignment.
- **Inputs:** Screen location; color; C-string text; font specification; justification (0=left, 1=center).
- **Outputs/Return:** None.
- **Side effects:** Activates font via `FontData.Use()`; sets foreground color and text pen position.
- **Calls:** `FontData.Use()`, `CopyCStringToPascal()`, `StringWidth()`, `SetColor()`, `MoveTo()`, `DrawString()`.
- **Notes:** Converts C string to Pascal string (Str255) with buffer limit (255 chars). Center justification halves string width and subtracts from x position. Invalid justification (default case) returns silently without drawing.

### set_path_drawing
- **Signature:** `void set_path_drawing(rgb_color& color)`
- **Purpose:** Initialize drawing state for a path trace sequence.
- **Inputs:** Color for the path.
- **Outputs/Return:** None.
- **Side effects:** Sets pen size to 1├ù1 and foreground color.
- **Calls:** `PenSize()`, `SetColor()`.
- **Notes:** Typically called once per path before multiple `draw_path()` calls.

### draw_path
- **Signature:** `void draw_path(short step, world_point2d& location)`
- **Purpose:** Draw a single segment of a path trace, either starting a new path or extending it.
- **Inputs:** Step index (0 = start, non-zero = continue); current location.
- **Outputs/Return:** None.
- **Side effects:** Sets Quickdraw pen position (MoveTo on step 0) or draws line segment (LineTo on step > 0).
- **Calls:** `MoveTo()` or `LineTo()`.
- **Notes:** Ternary branch: step==0 moves without drawing; step>0 draws line. Relies on prior `set_path_drawing()` call to configure pen.

## Control Flow Notes
These functions are called during the overhead map rendering frame. Typical sequence:
1. `set_path_drawing()` initializes pen once per path set.
2. Multiple `draw_path()` calls trace the path as a connected line.
3. `draw_polygon()` renders static map geometry.
4. `draw_line()` renders miscellaneous line segments (line-of-sight, etc.).
5. `draw_thing()` renders entities/objects.
6. `draw_player()` renders the player avatar.
7. `draw_text()` adds labels.

No explicit shutdown; Quickdraw state persists until next frame or caller resets.

## External Dependencies
- **Quickdraw API:** `RGBForeColor()`, `OpenPoly()`, `ClosePoly()`, `KillPoly()`, `MoveTo()`, `LineTo()`, `PenSize()`, `PaintRect()`, `FrameOval()`, `FillPoly()`, `FramePoly()`, `SetRect()`, `GetQDGlobalsBlack()`, `DrawString()`, `StringWidth()`, `CopyCStringToPascal()`
- **Parent class:** `OverheadMapClass` (via `OverheadMapRenderer.h`); provides `GetVertex(short)`, `translate_point2d()`, `normalize_angle()`
- **Types defined elsewhere:** `rgb_color`, `world_point2d`, `angle`, `FontSpecifier`
- **C Standard Library:** `<string.h>` for `strncpy()`
