# Source_Files/RenderOther/OverheadMap_SDL.cpp

## File Purpose
SDL-based concrete implementation of the OverheadMapClass for rendering game map elements (polygons, objects, players, paths, text) to the overhead map display. Provides the rendering backend for the Aleph One minimap/automap feature.

## Core Responsibilities
- Render filled/outlined polygons on the overhead map
- Draw line segments connecting map vertices
- Display game objects (scenery/items) as rectangles or octagons
- Render player position and facing direction as directional triangles
- Draw text annotations on the map
- Incrementally render player/unit paths as line chains
- Convert game world colors to SDL pixel values

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` | struct | RGB color specification (red, green, blue components) |
| `world_point2d` | struct | 2D world coordinate (x, y) |
| `angle` | typedef | Angular direction/facing |
| `FontSpecifier` | struct | Font info and styling for text rendering |
| `SDL_Rect` | struct (SDL) | Rectangle for SDL drawing operations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `draw_surface` | SDL_Surface* | extern | Target SDL surface for all drawing operations (from screen_sdl.cpp) |
| `vertex_array` | world_point2d* | static (draw_polygon) | Cached vertex buffer for polygon rendering; grown as needed |
| `max_vertices` | int | static (draw_polygon) | Tracks allocated size of `vertex_array` cache |
| `path_pixel` | uint32 | member | Cached SDL pixel value for path segments (set by `set_path_drawing`) |
| `path_point` | world_point2d | member | Persistent last-drawn point; state maintained across `draw_path()` calls |

## Key Functions / Methods

### draw_polygon
- **Signature:** `void draw_polygon(short vertex_count, short *vertices, rgb_color& color)`
- **Purpose:** Render a filled or outlined polygon on the overhead map.
- **Inputs:** `vertex_count` (number of vertices), `vertices` (array of map vertex indices), `color` (RGB color)
- **Outputs/Return:** void; modifies draw_surface
- **Side effects:** Dynamic memory allocation (realloc vertex_array if needed); renders to global `draw_surface`
- **Calls:** `GetVertex()` (inherited), `SDL_MapRGB()`, `::draw_polygon()`
- **Notes:** Static vertex buffer is reused and grown lazily. Color components are right-shifted by 8 bits before SDL mapping.

### draw_line
- **Signature:** `void draw_line(short *vertices, rgb_color &color, short pen_size)`
- **Purpose:** Draw a single line segment between two map vertices.
- **Inputs:** `vertices` (two-element array of vertex indices), `color` (RGB), `pen_size` (line width)
- **Outputs/Return:** void; modifies draw_surface
- **Side effects:** Renders to global `draw_surface`
- **Calls:** `GetVertex()`, `SDL_MapRGB()`, `::draw_line()`
- **Notes:** Always connects exactly vertices[0] and vertices[1].

### draw_thing
- **Signature:** `void draw_thing(world_point2d &center, rgb_color &color, short shape, short radius)`
- **Purpose:** Render a game object (scenery, item, etc.) at a given map location.
- **Inputs:** `center` (2D world position), `color` (RGB), `shape` (_rectangle_thing or _circle_thing), `radius` (size)
- **Outputs/Return:** void; modifies draw_surface
- **Side effects:** Renders to global `draw_surface`
- **Calls:** `SDL_MapRGB()`, `SDL_FillRect()` (for rectangles), `::draw_line()` (for octagons)
- **Notes:** Radius scaling: downsample factor 0.75├ùradius, upsample 1.5├ùradius. Circles are approximated as 8-sided octagons.

### draw_player
- **Signature:** `void draw_player(world_point2d &center, angle facing, rgb_color &color, short shrink, short front, short rear, short rear_theta)`
- **Purpose:** Render a player/unit as a directional triangle showing position and facing.
- **Inputs:** `center` (position), `facing` (angle), `color` (RGB), `shrink` (scale bit shift), `front`/`rear` (distances), `rear_theta` (rear spread angle)
- **Outputs/Return:** void; modifies draw_surface
- **Side effects:** Renders to global `draw_surface`
- **Calls:** `SDL_MapRGB()`, `translate_point2d()`, `normalize_angle()`, `::draw_polygon()`
- **Notes:** Constructs triangle: apex at `front` distance along `facing`, two rear vertices at `rear` distance ┬▒`rear_theta`. Distances are bit-shifted by `shrink`.

### draw_text
- **Signature:** `void draw_text(world_point2d &location, rgb_color &color, char *text, FontSpecifier& FontData, short justify)`
- **Purpose:** Render text annotation at a map location.
- **Inputs:** `location` (2D position), `color` (RGB), `text` (string), `FontData` (font spec), `justify` (alignment flag)
- **Outputs/Return:** void; modifies draw_surface
- **Side effects:** Renders to global `draw_surface`
- **Calls:** `SDL_MapRGB()`, `text_width()`, `::draw_text()`
- **Notes:** Supports center-justification (adjusts x by half text width) or left-justified. Color component right-shift by 8.

### set_path_drawing
- **Signature:** `void set_path_drawing(rgb_color &color)`
- **Purpose:** Cache the color for subsequent path-drawing calls.
- **Inputs:** `color` (RGB)
- **Outputs/Return:** void
- **Side effects:** Updates member variable `path_pixel`
- **Calls:** `SDL_MapRGB()`
- **Notes:** Must be called before any `draw_path()` calls to initialize the path color.

### draw_path
- **Signature:** `void draw_path(short step, world_point2d &location)`
- **Purpose:** Incrementally render a path as a continuous chain of line segments.
- **Inputs:** `step` (0 for first point, non-zero for subsequent), `location` (current path point)
- **Outputs/Return:** void
- **Side effects:** Updates member `path_point`; renders line to `draw_surface` if `step != 0`
- **Calls:** `::draw_line()` (if step != 0)
- **Notes:** Stateful: first call with step=0 initializes `path_point`; subsequent calls draw from prior point to current. Used to trace player movements or movement predictions.

## Control Flow Notes
This file implements rendering callbacks invoked by the overhead map system each frame. The typical draw sequence is:
1. `set_path_drawing()` (once per frame)
2. Multiple calls to `draw_polygon()`, `draw_line()` (map geometry)
3. Multiple calls to `draw_thing()` (objects), `draw_player()` (units)
4. Sequential calls to `draw_path()` (player paths)
5. Multiple calls to `draw_text()` (annotations)

Not directly tied to init/update/render pipeline; rather, called on-demand during minimap render.

## External Dependencies
- **SDL library:** `SDL_Surface`, `SDL_Rect`, `SDL_MapRGB()`, `SDL_FillRect()`
- **Global symbols (defined elsewhere):** 
  - `draw_surface` (from screen_sdl.cpp)
  - `::draw_polygon()`, `::draw_line()`, `::draw_text()` (from screen_drawing module)
  - `GetVertex()` (inherited from OverheadMapClass base)
  - `translate_point2d()`, `normalize_angle()` (geometry utilities)
  - `text_width()` (font utility from sdl_fonts)
