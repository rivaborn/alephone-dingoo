# Source_Files/RenderOther/sdl_fonts.h

## File Purpose
Defines the font rendering interface for the Aleph One game engine, supporting both bitmap fonts (`sdl_font_info`) and TrueType fonts (`ttf_font_info`) via SDL. Provides abstractions for text drawing, measurement, and styling across different font backends.

## Core Responsibilities
- Define abstract `font_info` interface for font metrics and text rendering
- Implement bitmap font rendering with kerning and styling support
- Implement TrueType font rendering (conditional on `HAVE_SDL_TTF`)
- Provide text measurement, truncation, and styled text operations
- Manage font resource lifecycle (loading/unloading via `LoadedResource`)
- Support multi-style rendering (bold, italic, underline, etc.)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `font_info` | Abstract class | Base interface for all font implementations; defines metrics getters and text operations |
| `sdl_font_info` | Class | Bitmap font implementation storing pixmap, character tables, and layout metrics |
| `ttf_font_info` | Class | TrueType font implementation wrapping SDL_ttf; manages multiple style variants |
| `ttf_font_key_t` | typedef (tuple) | Cache key: `<font_path, size, kerning_adjustment>` |

## Global / File-Static State
None.

## Key Functions / Methods

### draw_text
- Signature: `int draw_text(SDL_Surface *s, const char *text, size_t length, int x, int y, uint32 pixel, uint16 style, bool utf8 = false) const`
- Purpose: Render plaintext at position (x, y) in given style
- Inputs: Surface, text buffer, pixel color, style flags, UTF-8 mode
- Outputs/Return: Pixel width of rendered text
- Side effects: Modifies SDL_Surface
- Calls: `_draw_text()` (virtual, delegates to subclass)
- Notes: UTF-8 support optional; dispatches to subclass implementation

### text_width
- Signature: `uint16 text_width(const char *text, size_t length, uint16 style, bool utf8 = false) const` and overload without length
- Purpose: Measure text width in pixels for given style
- Inputs: Text buffer, style flags, UTF-8 mode
- Outputs/Return: Width in pixels
- Side effects: None
- Calls: `_text_width()` (virtual)

### draw_styled_text
- Signature: `int draw_styled_text(SDL_Surface *s, const std::string& text, size_t length, int x, int y, uint32 pixel, uint16 initial_style, bool utf = false) const`
- Purpose: Render text with embedded style codes (e.g., mid-string bold/italic changes)
- Inputs: Surface, styled text string, position, pixel color
- Outputs/Return: Pixel width
- Side effects: Modifies SDL_Surface
- Calls: Parses style markers and calls `_draw_text()` per segment

### char_width
- Signature: `int8 char_width(uint8 c, uint16 style) const` (virtual in base, implemented in subclasses)
- Purpose: Get kerning-adjusted width of a single character
- Inputs: Character code, style flags
- Outputs/Return: Width (may be negative for kerning)
- Side effects: None
- Notes: Pure virtual in base; subclasses use character/width tables

### trunc_text / trunc_styled_text
- Signature: `int trunc_text(const char *text, int max_width, uint16 style) const`
- Purpose: Truncate text to fit within max_width pixels, returning character count
- Inputs: Text buffer, pixel budget, style
- Outputs/Return: Number of characters that fit
- Side effects: None
- Calls: `_trunc_text()` (virtual)

### style_at
- Signature: `std::string style_at(const std::string& text, std::string::const_iterator pos, uint16 style) const`
- Purpose: Query active style at a position in styled text
- Inputs: Styled text, iterator position, base style
- Outputs/Return: Style string representation at that point
- Side effects: None

### get_ascent, get_descent, get_height, get_line_height
- Purpose: Font metric accessors (pure virtual in base, implemented in subclasses)
- Outputs/Return: Metric in pixels

## Control Flow Notes
- **Initialization**: `initialize_fonts()` called at engine startup to register available fonts
- **Loading**: `load_font(TextSpec)` instantiates appropriate font class (bitmap or TTF based on resource availability)
- **Rendering**: Per-frame, game calls `draw_text()` or `draw_styled_text()` on UI elements; virtual dispatch selects bitmap or TTF path
- **Cleanup**: `unload_font()` triggers virtual `_unload()`, which deletes the font object via RAII in `LoadedResource`
- **Conditional paths**: TTF support guarded by `#ifdef HAVE_SDL_TTF`; bitmap fonts always available

## External Dependencies
- **FileHandler.h** ΓÇô `LoadedResource` class for resource lifecycle management
- **SDL_ttf.h** (conditional) ΓÇô TTF rendering; `TTF_Font`, `TTF_FontAscent()`, etc.
- **boost/tuple/tuple.hpp** (conditional) ΓÇô `ttf_font_key_t` tuple template
- **string** (std) ΓÇô styled text strings
- Undefined symbols used: `TextSpec` (assumed defined elsewhere in render/UI headers), `uint16`, `uint32`, `int16`, `int8`, `uint8` (fixed-width types)
