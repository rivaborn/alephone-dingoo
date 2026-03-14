# Source_Files/RenderOther/screen_drawing.cpp

## File Purpose
Low-level SDL-based screen drawing implementation for the Aleph One game engine. Provides services for rendering shapes, text, geometric primitives, and managing the drawing surface, clipping, colors, and fonts for both in-game HUD and menus.

## Core Responsibilities
- Manage drawing surfaces (screen, world buffer, HUD buffer, terminal buffer) via port switching
- Render 2D shapes and sprites with coordinate translation and clipping
- Render text using bitmap fonts (sdl_font_info) and TrueType fonts (ttf_font_info)
- Draw geometric primitives: filled/outlined rectangles, lines (with Cohen-Sutherland clipping), and filled convex polygons
- Maintain interface rectangle definitions, color palettes, and font specifications
- Parse interface rectangles and colors from XML configuration
- Support text layout: wrapping, horizontal/vertical centering, right/top justification

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `screen_rectangle` | struct | Portable rectangle (top, left, bottom, right) for UI layout |
| `FontSpecifier` | class | Font specification with name, size, style, optional path |
| `rgb_color` | struct | 16-bit RGB color (red, green, blue) |
| `sdl_font_info` | class | Bitmap font with glyph metrics, pixmap, and rendering methods |
| `ttf_font_info` | class | TrueType font with rendering via SDL_TTF |
| `world_point2d` | struct | 2D point for line/polygon clipping and drawing |
| `XML_RectangleParser` | class | XML element parser for rectangle definitions |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `interface_rectangles` | `screen_rectangle[20]` | static | Predefined UI layout rectangles (buttons, display areas) |
| `InterfaceFonts` | `FontSpecifier[7]` | static | Font specifications for UI text rendering |
| `InterfaceColors` | `rgb_color[26]` | static | Hardcoded color palette for interface and player colors |
| `draw_surface` | `SDL_Surface*` | static | Current target surface for drawing operations |
| `old_draw_surface` | `SDL_Surface*` | static | Saved surface for port restoration |
| `draw_clip_rect_active` | `bool` | static | Flag indicating if clipping is active |
| `draw_clip_rect` | `screen_rectangle` | static | Current clipping bounds |
| `RectangleParser` | `XML_RectangleParser` | static | Singleton parser instance for XML rectangles |

## Key Functions / Methods

### initialize_screen_drawing
- Signature: `void initialize_screen_drawing(void)`
- Purpose: One-time initialization of screen drawing subsystem
- Inputs: None
- Outputs/Return: None
- Side effects: Loads interface rectangles and colors; initializes all fonts via `FontSpecifier::Init()`
- Calls: `load_interface_rectangles()`, `load_screen_interface_colors()`, `FontSpecifier::Init()`
- Notes: Must be called before any drawing operations

### _set_port_to_screen_window / _set_port_to_gworld / _set_port_to_HUD / _set_port_to_term
- Signature: `void _set_port_to_screen_window(void)` (and variants)
- Purpose: Switch rendering target surface (save old surface)
- Inputs: None
- Outputs/Return: None
- Side effects: Updates `draw_surface` and `old_draw_surface`
- Calls: `SDL_GetVideoSurface()` or direct buffer reference
- Notes: Asserts if called while a port is already set; must pair with `_restore_port()`

### _restore_port
- Signature: `void _restore_port(void)`
- Purpose: Restore previous drawing surface
- Inputs: None
- Outputs/Return: None
- Side effects: Restores `draw_surface` to saved value; clears `old_draw_surface`
- Calls: None
- Notes: Must be called after corresponding `_set_port_to_*()` call

### set_drawing_clip_rectangle
- Signature: `void set_drawing_clip_rectangle(short top, short left, short bottom, short right)`
- Purpose: Set or disable clipping bounds for all drawing operations
- Inputs: `top, left, bottom, right` ΓÇö clip rectangle bounds; top < 0 disables clipping
- Outputs/Return: None
- Side effects: Updates `draw_clip_rect_active` flag and `draw_clip_rect`
- Calls: None

### _draw_screen_shape
- Signature: `void _draw_screen_shape(shape_descriptor shape_id, screen_rectangle *destination, screen_rectangle *source)`
- Purpose: Render shape sprite with source/destination clipping
- Inputs: `shape_id` ΓÇö shape descriptor; `destination` ΓÇö target rectangle; `source` ΓÇö source sub-region (NULL = entire shape)
- Outputs/Return: None
- Side effects: Retrieves shape surface, blits to `draw_surface`, updates screen if drawing to video surface
- Calls: `get_shape_surface()`, `SDL_BlitSurface()`, `SDL_UpdateRects()`, `SDL_FreeSurface()`
- Notes: Handles 8-bit surface format conversion via `SDL_DisplayFormat()`; performs format-safe depth conversion

### draw_glyph (template)
- Signature: `template <class T> inline static int draw_glyph(uint8 c, int x, int y, T *p, int pitch, int clip_left, int clip_top, int clip_right, int clip_bottom, uint32 pixel, const sdl_font_info *font, bool oblique)`
- Purpose: Render single glyph to pixel buffer with clipping and optional italics
- Inputs: Character `c`, position `(x,y)`, pixel buffer and pitch, clip bounds, pixel value, font, oblique flag
- Outputs/Return: Glyph advance width (in pixels)
- Side effects: Writes pixel values directly to buffer
- Calls: None (direct pixel manipulation)
- Notes: Per-pixel rendering loop; handles kerning, ascent, and oblique offset; bounds-checks all edges

### draw_text (template)
- Signature: `template <class T> inline static int draw_text(const uint8 *text, size_t length, int x, int y, T *p, int pitch, int clip_left, int clip_top, int clip_right, int clip_bottom, uint32 pixel, const sdl_font_info *font, uint16 style)`
- Purpose: Render text string with styling (bold, underline, italic)
- Inputs: Text buffer and length, position, pixel buffer, clip bounds, font, style flags
- Outputs/Return: Total rendered width
- Side effects: Calls `draw_glyph()` for each character; writes pixels to buffer
- Calls: `draw_glyph()` (for each character; up to 3├ù per char if bold/underline)
- Notes: Skips characters outside font range; implements bold via second pass offset by 1 pixel; underline drawn as horizontal lines

### sdl_font_info::_draw_text
- Signature: `int sdl_font_info::_draw_text(SDL_Surface *s, const char *text, size_t length, int x, int y, uint32 pixel, uint16 style, bool) const`
- Purpose: High-level bitmap font text rendering to SDL surface
- Inputs: Target surface, text, length, position, pixel color, style, UTF-8 flag (unused for bitmap fonts)
- Outputs/Return: Text width rendered
- Side effects: Locks/unlocks SDL surface; calls dispatch based on pixel depth; updates screen rect if on video surface
- Calls: `draw_text()` (template), `SDL_LockSurface()`, `SDL_UnlockSurface()`, `SDL_UpdateRect()`
- Notes: Handles 1/2/4 bytes-per-pixel depth; calculates clip rect from `draw_clip_rect_active`

### ttf_font_info::_draw_text
- Signature: `int ttf_font_info::_draw_text(SDL_Surface *s, const char *text, size_t length, int x, int y, uint32 pixel, uint16 style, bool utf8) const`
- Purpose: TrueType font text rendering (requires SDL_TTF)
- Inputs: Target surface, text, length, position, pixel color, style, UTF-8 flag
- Outputs/Return: Rendered text width
- Side effects: Renders text to temporary surface via SDL_TTF; blits to target; frees temp surface; updates screen
- Calls: `process_printable()`, `process_macroman()`, `TTF_RenderUTF8_Blended/Solid()`, `TTF_RenderUNICODE_Blended/Solid()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`, `SDL_UpdateRect()`
- Notes: Respects `environment_preferences->smooth_text` for blended vs. solid rendering; clips output to `draw_clip_rect` if active

### _draw_screen_text
- Signature: `void _draw_screen_text(const char *text, screen_rectangle *destination, short flags, short font_id, short text_color)`
- Purpose: High-level screen text rendering with layout (wrapping, justification, centering)
- Inputs: Text string, destination rectangle, layout flags, font index, color index
- Outputs/Return: None
- Side effects: May recursively call self if text wrapping occurs; modifies flags to disable vertical centering on wrap
- Calls: `InterfaceFonts[].Info->text_width()`, `InterfaceFonts[].Info->trunc_text()`, `_draw_screen_text()` (recursive), `draw_text()`, `_get_interface_color()`
- Notes: Text wrapping breaks at spaces; horizontal position calculated before drawing; vertical position accounts for font height and baseline

### draw_line
- Signature: `void draw_line(SDL_Surface *s, const world_point2d *v1, const world_point2d *v2, uint32 pixel, int pen_size)`
- Purpose: Draw line segment with thickness (pen size) and clipping
- Inputs: Surface, start/end points, pixel color, pen size
- Outputs/Return: None
- Side effects: Locks/unlocks surface; renders thin line via DDA + Cohen-Sutherland or thick line via polygon
- Calls: `cs_code()`, `draw_thin_line_noclip()`, `draw_polygon()` (for thick lines), `SDL_LockSurface()`, `SDL_UnlockSurface()`
- Notes: Pen size 1 uses DDA line algorithm with Cohen-Sutherland clipping; larger pens convert line to hexagon and fill via `draw_polygon()`

### draw_polygon
- Signature: `void draw_polygon(SDL_Surface *s, const world_point2d *vertex_array, int vertex_count, uint32 pixel)`
- Purpose: Render clipped, filled, convex polygon
- Inputs: Surface, vertex array, count, pixel color
- Outputs/Return: None
- Side effects: Allocates/reallocates temporary vertex buffers (static); clips via Sutherland-Hodgman; fills via span rasterization
- Calls: Macro expansions for clip operations; `SDL_FillRect()`, `SDL_UpdateRect()`
- Notes: Clips on all four edges; uses fixed-point (16.16) DDA for span edge evaluation; fills top-to-bottom

### _fill_rect / _frame_rect
- Signature: `void _fill_rect(screen_rectangle *rectangle, short color_index)` / `void _frame_rect(screen_rectangle *rectangle, short color_index)`
- Purpose: Fill or outline a rectangle with interface color
- Inputs: Rectangle, color index
- Outputs/Return: None
- Side effects: Calls `SDL_FillRect()`, updates video surface if drawing to screen
- Calls: `_get_interface_color()`, `SDL_MapRGB()`, `SDL_FillRect()`, `SDL_UpdateRects()` / `draw_rectangle()`
- Notes: `_fill_rect` with NULL rectangle erases entire surface

### get_interface_rectangle / get_interface_color / get_interface_font
- Signature: `screen_rectangle *get_interface_rectangle(short index)` / `const rgb_color &get_interface_color(short index)` / `FontSpecifier &get_interface_font(short index)`
- Purpose: Accessor functions for UI layout data
- Inputs: Index
- Outputs/Return: Pointer/reference to rectangle/color/font
- Side effects: None
- Calls: None
- Notes: All include bounds checking via assert

## Control Flow Notes
1. **Initialization phase** (`initialize_screen_drawing`): Called once at startup; sets up static data (rectangles, colors, fonts).
2. **Per-frame drawing**: Code calls `_set_port_to_*` to switch rendering target, draws primitives/text, then calls `_restore_port`.
3. **Text rendering pipeline**: High-level `_draw_screen_text` -> `draw_text` template -> per-glyph `draw_glyph` template -> pixel-level writes. Alternate path for TTF via SDL_TTF.
4. **Clipping**: `set_drawing_clip_rectangle` configures bounds; individual drawing functions check bounds or apply Cohen-Sutherland/Sutherland-Hodgman algorithms.
5. **SDL integration**: Surface locking/unlocking brackets pixel-level operations; screen updates called selectively only when drawing to video surface.

## External Dependencies
- **SDL library**: `SDL_Surface`, `SDL_Rect`, `SDL_BlitSurface`, `SDL_FillRect`, `SDL_UpdateRects`, `SDL_GetVideoSurface`, `SDL_LockSurface`, `SDL_UnlockSurface`, `SDL_DisplayFormat`, `SDL_MapRGB`, `SDL_FreeSurface`
- **SDL_TTF**: `TTF_RenderUTF8_Blended`, `TTF_RenderUTF8_Solid`, `TTF_RenderUNICODE_Blended`, `TTF_RenderUNICODE_Solid`, `TTF_FontAscent`, `TTF_FontHeight`
- **Game engine headers**:
  - `shape_descriptors.h` ΓÇö shape descriptor decoding
  - `sdl_fonts.h` ΓÇö font abstraction (`sdl_font_info`, `ttf_font_info`, `font_info` base)
  - `ColorParser.h`, `FontHandler.h` ΓÇö XML configuration support
  - `map.h`, `interface.h`, `shell.h`, `screen.h` ΓÇö game state and callbacks
  - `fades.h` ΓÇö screen effects (included but not directly used in this file)
