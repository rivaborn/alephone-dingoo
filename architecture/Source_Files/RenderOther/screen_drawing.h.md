# Source_Files/RenderOther/screen_drawing.h

## File Purpose
Header file defining the HUD/UI rendering interface for a Marathon-like first-person shooter. Provides screen drawing primitives, text rendering, and shape display functions, along with enumerated constants for UI rectangles, colors, fonts, and text justification options. Supports both legacy and SDL rendering backends.

## Core Responsibilities
- Define UI layout constants (rectangle IDs for HUD elements, buttons)
- Manage rendering target surfaces (screen window, offscreen buffer, HUD buffer, terminal)
- Provide high-level drawing functions for shapes, text, and rectangles
- Expose text measurement utilities for layout and wrapping
- Parse XML configuration for interface rectangle and color definitions
- Supply font and color palette management for UI rendering
- Offer SDL-specific inline wrappers for text and primitive drawing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `screen_rectangle` | struct | Portable rectangle definition (top, left, bottom, right) matching platform Rect format |
| Rectangle ID enum | enum | Named constants for game HUD and interface button rectangles (player name, oxygen, weapon display, menu buttons, etc.) |
| Color enum | enum | Palette indices for interface and game UI colors (energy, inventory, computer interface, etc.) |
| Text flags enum | enum | Justification and formatting flags for text drawing (center, wrap, top/bottom-justified) |
| Font enum | enum | Interface font type selector (weapon name, player name, computer interface, net stats) |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_screen_drawing
- Signature: `void initialize_screen_drawing(void)`
- Purpose: Initialize screen drawing system, allocate buffers, load fonts/colors
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates memory, initializes global rendering state
- Calls: (definition elsewhere)
- Notes: Called at engine startup

### _set_port_to_screen_window / _set_port_to_gworld / _restore_port
- Signature: `void _set_port_to_screen_window(void)`, `void _set_port_to_gworld(void)`, `void _restore_port(void)`
- Purpose: Redirect drawing operations to different render targets (main screen, offscreen buffer, or saved port state)
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies active rendering target; subsequent draw calls use new target
- Calls: (implementation specific)
- Notes: Classic Mac Toolbox pattern; supports double-buffering

### _draw_screen_shape
- Signature: `void _draw_screen_shape(shape_descriptor shape_id, screen_rectangle *destination, screen_rectangle *source)`
- Purpose: Draw a sprite/shape from collection to screen with optional source clipping
- Inputs: shape_id (packed collection/clut/shape bits), destination rectangle, source rectangle (or NULL for full sprite)
- Outputs/Return: None
- Side effects: Renders to active port
- Calls: (visible in implementation file)
- Notes: Source clipping useful for partial sprite display

### _draw_screen_text
- Signature: `void _draw_screen_text(const char *text, screen_rectangle *destination, short flags, short font_id, short text_color)`
- Purpose: Draw text with justification, wrapping, and color
- Inputs: text string, bounding rectangle, justification flags, font index, color palette index
- Outputs/Return: None
- Side effects: Renders to active port
- Calls: Font system, text measurement functions
- Notes: Flags control centering, wrapping, and alignment within rectangle

### _text_width
- Signature: `short _text_width(const char *buffer, short font_id)`, `short _text_width(const char *buffer, int start, int length, short font_id)`
- Purpose: Measure text width for layout calculations
- Inputs: text buffer, font index, optional substring range
- Outputs/Return: Width in pixels
- Side effects: None
- Calls: Font system
- Notes: Supports full string or substring measurement

### _erase_screen / _fill_screen_rectangle / _frame_rect
- Signature: `void _erase_screen(short color_index)`, `void _fill_screen_rectangle(screen_rectangle *rectangle, short color_index)`, `void _frame_rect(screen_rectangle *rectangle, short color_index)`
- Purpose: Clear screen or draw solid/outline rectangles
- Inputs: Color palette index (and rectangle bounds for fill/frame)
- Outputs/Return: None
- Side effects: Renders to active port
- Calls: (primitive rendering)
- Notes: Basic drawing primitives for HUD backgrounds and borders

### get_interface_rectangle / get_interface_color / get_interface_font
- Signature: `screen_rectangle *get_interface_rectangle(short index)`, `const rgb_color &get_interface_color(short index)`, `FontSpecifier &get_interface_font(short index)`
- Purpose: Retrieve layout, color, and font configuration by index
- Inputs: Rectangle/color/font ID enum value
- Outputs/Return: Pointer/reference to configured element
- Side effects: None
- Calls: (definition elsewhere, likely loaded from XML)
- Notes: Supports dynamic UI configuration from XML parsing

### InterfaceRectangles_GetParser
- Signature: `XML_ElementParser *InterfaceRectangles_GetParser()`
- Purpose: Retrieve XML parser object for interface rectangle configuration
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser subclass
- Side effects: None
- Calls: (definition elsewhere)
- Notes: Enables XML-driven UI layout

### SDL-specific inline helpers (draw_text, char_width, text_width, trunc_text, draw_polygon, draw_line, draw_rectangle)
- Purpose: Thin wrappers around `font_info` and SDL drawing functions for conditional compilation
- Inputs: SDL_Surface target, coordinates, text/geometry data, pixel color, font/style info
- Outputs/Return: Measurement or void
- Side effects: Render to surface
- Calls: Delegated to `font_info` methods or SDL primitives
- Notes: Null-checked; return 0 if font unavailable; allow size_t overloads for bounds-safe operation

## Control Flow Notes
This file is part of the **rendering pipeline** that executes during frame rendering:
1. **Initialization phase**: `initialize_screen_drawing()` sets up buffers, loads colors/fonts from XML
2. **Frame rendering phase**: HUD code calls `_set_port_to_*()` to select render target, then calls `_draw_screen_shape()`, `_draw_screen_text()`, etc.
3. **Post-frame**: `_restore_port()` or screen buffer swap commits HUD to display

XML parsing of interface layout happens asynchronously or at load time via `InterfaceRectangles_GetParser()`, decoupling hardcoded layout from rendering code.

## External Dependencies
- **XML_ElementParser.h**: XML parsing framework for dynamic interface configuration
- **shape_descriptors.h**: Shape ID encoding (collection, CLUT, shape bits)
- **sdl_fonts.h**: Font info class and SDL font loading
- **SDL (conditional)**: `SDL_Surface`, `SDL_Rect`, `SDL_ttf` for rendering and text drawing
- **Implicit platform abstractions**: Port/GWorld concepts (classic Mac Toolbox patterns)
