# Source_Files/RenderOther/sdl_fonts.cpp

## File Purpose
Manages font loading, caching, and text measurement for the Aleph One engine. Supports both bitmap fonts (Mac-style resource format) and TrueType fonts via SDL_ttf, with reference counting and styled text parsing (bold, italic, plain).

## Core Responsibilities
- Initialize font resources from disk (bitmap "Fonts" resources or TTF files)
- Load and cache bitmap fonts (`sdl_font_info`) with reference counting
- Load and cache TrueType fonts (`ttf_font_info`) with multiple style variants (normal, bold, italic, bold+italic)
- Parse and apply inline style codes (`|b`, `|i`, `|p`) in text strings
- Measure text width considering font metrics, style, and shadow effects
- Truncate text to fit within max width constraints
- Handle Mac Roman Γåö Unicode encoding for TTF text
- Unload fonts with safe deallocation via reference counting

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `sdl_font_info` | class (implicit) | Bitmap font: pixel data, metrics tables, character dimensions |
| `ttf_font_info` | class (implicit) | TrueType font: holds 4 style variants, adjustment height, font keys |
| `font_info` | class (implicit base) | Virtual interface for text measurement and rendering |
| `id_and_size_t` | typedef (pair) | Key tuple: (font resource ID, size in points) |
| `font_list_t` | typedef (map) | Cache: maps (ID, size) ΓåÆ loaded bitmap font info |
| `ttf_font_list_t` | typedef (map) | Cache: maps TTF key ΓåÆ (TTF_Font*, ref_count) |
| `ref_counted_ttf_font_t` | typedef (pair) | (TTF_Font* pointer, ref count integer) |
| `style_separator` | class | Boost tokenizer functor; splits text on style codes (`\|[bipcrs]`) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `font_list` | `font_list_t` | static | Bitmap font cache; populated by `load_sdl_font()`, cleaned by `sdl_font_info::_unload()` |
| `ttf_font_list` | `ttf_font_list_t` | static (HAVE_SDL_TTF) | TTF font cache; populated by `load_ttf_font()` |
| `data_search_path` | `vector<DirectorySpecifier>` | extern (shell_sdl.cpp) | Directories to search for font resource files |

## Key Functions / Methods

### initialize_fonts()
- Signature: `void initialize_fonts(void)`
- Purpose: Entry point; opens font resource files from `data_search_path`
- Inputs: None (uses global `data_search_path`)
- Outputs/Return: None
- Side effects: Opens resource files ("Fonts" or "Fonts.fntA"); calls `fix_missing_*` functions if SDL_ttf available; calls `exit(1)` if no fonts found and SDL_ttf unavailable
- Calls: `open_res_file()`, `fix_missing_overhead_map_fonts()`, `fix_missing_interface_fonts()`
- Notes: Searches directories iteratively; tolerates missing files in some paths

### load_sdl_font(const TextSpec &spec)
- Signature: `sdl_font_info *load_sdl_font(const TextSpec &spec)`
- Purpose: Load bitmap font (NFNT/FONT resource) and associated metrics; cache by (ID, size)
- Inputs: `spec` ΓÇö font ID, size
- Outputs/Return: Pointer to `sdl_font_info` or NULL if not found
- Side effects: Allocates `sdl_font_info` and pixmap buffer; updates `font_list` cache; increments ref_count on hit
- Calls: `get_resource()`, `SDL_RWFromMem()`, `SDL_RW*` I/O functions, `byte_swap_memory()`
- Notes: Expands 1-bit bitmap to 1-byte-per-pixel (0x00/0xff); sets up location and width tables; asserts pixmap allocation

### load_ttf_font(const std::string& path, uint16 style, int16 size) [HAVE_SDL_TTF]
- Signature: `static TTF_Font *load_ttf_font(const std::string& path, uint16 style, int16 size)`
- Purpose: Load TTF font file (or built-in "mono"), cache with ref counting
- Inputs: `path` ΓÇö file path or "mono"; `style` ΓÇö styleBold/styleItalic flags; `size` ΓÇö points
- Outputs/Return: `TTF_Font*` pointer or NULL on failure
- Side effects: Opens TTF file; increments ref count on cache hit; updates `ttf_font_list`; sets TTF style and hinting via SDL_ttf
- Calls: `TTF_OpenFont()`, `TTF_OpenFontRW()`, `TTF_SetFontStyle()`, `TTF_SetFontHinting()`
- Notes: Special case for "mono" ΓåÆ uses embedded `aleph_sans_mono_bold` data; hinting depends on `environment_preferences->smooth_text`

### load_font(const TextSpec &spec)
- Signature: `font_info *load_font(const TextSpec &spec)`
- Purpose: Main public entry point; prioritizes TTF, falls back to bitmap fonts
- Inputs: `spec` ΓÇö TextSpec with TTF paths (normal, bold, oblique, bold_oblique) or bitmap font ID
- Outputs/Return: Pointer to `ttf_font_info` or `sdl_font_info` cast to `font_info*`, or NULL
- Side effects: If TTF paths exist, loads up to 4 style variants with cascading fallbacks (e.g., if bold fails, try applying bold style to normal font); otherwise loads bitmap font
- Calls: `locate_font()`, `load_ttf_font()`, `load_sdl_font()`
- Notes: Asserts succeed on fallback attempts; returns 0 if spec.font == -1 and no TTF paths

### locate_font(const std::string& path) [HAVE_SDL_TTF]
- Signature: `static const char *locate_font(const std::string& path)`
- Purpose: Resolve font file path; special case "mono" and "" (built-in)
- Inputs: `path` ΓÇö file path string
- Outputs/Return: C string pointer (valid for lifetime of static buffer)
- Side effects: Sets static FileSpecifier; calls `SetNameWithPath()`
- Calls: `FileSpecifier::SetNameWithPath()`, `FileSpecifier::GetPath()`
- Notes: Returns "" if file not found

### sdl_font_info::_unload()
- Signature: `void sdl_font_info::_unload()`
- Purpose: Decrement ref count; delete and erase from cache if count Γëñ 0
- Inputs: None (operates on `this`)
- Outputs/Return: None
- Side effects: Decrements `ref_count`; calls `delete this` and erases from `font_list` on final unload
- Calls: `font_list.erase()`
- Notes: Searches `font_list` for matching pointer; deletes font data (pixmap, tables)

### ttf_font_info::_unload() [HAVE_SDL_TTF]
- Signature: `void ttf_font_info::_unload()`
- Purpose: Decrement ref counts for all 4 style variants; close TTF fonts; delete this
- Inputs: None (operates on `this`)
- Outputs/Return: None
- Side effects: Iterates `m_keys[0..styleUnderline-1]`, decrements each in `ttf_font_list`, closes via `TTF_CloseFont()`, erases from cache, nulls `m_styles`
- Calls: `TTF_CloseFont()`
- Notes: Unconditionally deletes `this` after cleanup

### unload_font(font_info *info)
- Signature: `void unload_font(font_info *info)`
- Purpose: Public unload wrapper; calls virtual `_unload()`
- Inputs: `info` ΓÇö pointer to font_info subclass
- Outputs/Return: None
- Side effects: Calls `info->_unload()` (virtual dispatch)
- Notes: Caller must not use pointer after call

### sdl_font_info::char_width(uint8 c, uint16 style)
- Signature: `int8 sdl_font_info::char_width(uint8 c, uint16 style) const`
- Purpose: Look up glyph width from bitmap font width table; add 1 if bold
- Inputs: `c` ΓÇö character code; `style` ΓÇö style flags (checks styleBold)
- Outputs/Return: Width in pixels, or 0 if character out of range, or fallback to last_character+1 if -1 (missing)
- Calls: None
- Notes: Returns 0 for c < first_character or c > last_character; if width == -1, uses last_character+1

### sdl_font_info::_text_width(const char *text, uint16 style, bool) [2 overloads]
- Signature: `uint16 sdl_font_info::_text_width(const char *text, uint16 style, bool) const`; `uint16 sdl_font_info::_text_width(const char *text, size_t length, uint16 style, bool) const`
- Purpose: Sum widths of all characters in null-terminated or length-delimited string
- Inputs: `text` ΓÇö string; `length` (overload 2); `style` ΓÇö style flags; `bool` ΓÇö unused parameter
- Outputs/Return: Total width as uint16 (asserts no overflow)
- Calls: `char_width()`
- Notes: Asserts width ΓëÑ 0 and fits in uint16

### sdl_font_info::_trunc_text(const char *text, int max_width, uint16 style)
- Signature: `int sdl_font_info::_trunc_text(const char *text, int max_width, uint16 style) const`
- Purpose: Count characters that fit within max_width
- Inputs: `text` ΓÇö string; `max_width` ΓÇö pixel limit; `style` ΓÇö style flags
- Outputs/Return: Number of characters that fit (stops at overflow or null terminator)
- Calls: `char_width()`
- Notes: Returns count, not pointer

### ttf_font_info::char_width(uint8 c, uint16 style) [HAVE_SDL_TTF]
- Signature: `int8 ttf_font_info::char_width(uint8 c, uint16 style) const`
- Purpose: Look up glyph advance width from TTF metrics via `TTF_GlyphMetrics()`
- Inputs: `c` ΓÇö Mac Roman character; `style` ΓÇö unused (font already set to style)
- Outputs/Return: Advance width in pixels
- Calls: `get_ttf()`, `mac_roman_to_unicode()`, `TTF_GlyphMetrics()`
- Notes: Converts Mac Roman to Unicode for glyph lookup

### ttf_font_info::_text_width [2 overloads] [HAVE_SDL_TTF]
- Signature: `uint16 ttf_font_info::_text_width(const char *text, uint16 style, bool utf8) const`; length variant
- Purpose: Measure text width using SDL_ttf; supports UTF-8 or Mac Roman
- Inputs: `text` ΓÇö string; `length` (overload 2); `style` ΓÇö style (passed to `get_ttf()`); `utf8` ΓÇö encoding flag
- Outputs/Return: Width in pixels
- Side effects: Allocates temporary buffers (static, max 1024 chars)
- Calls: `get_ttf()`, `process_printable()`, `process_macroman()`, `TTF_SizeUTF8()`, `TTF_SizeUNICODE()`
- Notes: Converts text via process_* helpers before measuring

### ttf_font_info::_trunc_text(const char *text, int max_width, uint16 style) [HAVE_SDL_TTF]
- Signature: `int ttf_font_info::_trunc_text(const char *text, int max_width, uint16 style) const`
- Purpose: Binary search (truncate-from-end loop) to fit text in max_width
- Inputs: `text` ΓÇö string; `max_width` ΓÇö pixel limit; `style` ΓÇö style
- Outputs/Return: Number of characters that fit
- Side effects: Uses static temp buffer (max 1024 chars)
- Calls: `get_ttf()`, `mac_roman_to_unicode()`, `TTF_SizeUNICODE()`, `strlen()`
- Notes: Optimistic path returns early if text fits; then loops decrementing char count

### ttf_font_info::process_printable(const char *src, int len) [HAVE_SDL_TTF]
- Signature: `char *ttf_font_info::process_printable(const char *src, int len) const`
- Purpose: Filter printable chars (>= space) from UTF-8 string into static buffer
- Inputs: `src` ΓÇö UTF-8 string; `len` ΓÇö max chars to process
- Outputs/Return: Pointer to static buffer (max 1024 chars, null-terminated)
- Notes: Skips control chars; clamps len to 1023

### ttf_font_info::process_macroman(const char *src, int len) [HAVE_SDL_TTF]
- Signature: `uint16 *ttf_font_info::process_macroman(const char *src, int len) const`
- Purpose: Convert Mac Roman string to Unicode (uint16 array); filter non-printable and convert tabs to spaces
- Inputs: `src` ΓÇö Mac Roman string; `len` ΓÇö max chars
- Outputs/Return: Pointer to static uint16 buffer (null-terminated, max 1024 entries)
- Calls: `mac_roman_to_unicode()`
- Notes: Skips control chars except tab (ΓåÆ space); clamps len to 1023

### font_info::text_width(const char *text, uint16 style, bool utf8) [2 overloads]
- Signature: `uint16 font_info::text_width(const char *text, uint16 style, bool utf8) const`; length variant
- Purpose: Public wrapper; dispatches to `_text_width()` and adds 1 pixel for shadow if styleShadow set
- Inputs: `text` ΓÇö string; `length` (overload 2); `style` ΓÇö style flags; `utf8` ΓÇö encoding
- Outputs/Return: Width (including shadow offset)
- Calls: `_text_width()`
- Notes: Virtual dispatch to sdl_font_info or ttf_font_info implementation

### font_info::draw_text(SDL_Surface *s, const char *text, size_t length, int x, int y, uint32 pixel, uint16 style, bool utf8)
- Signature: `int font_info::draw_text(...) const`
- Purpose: Render text to SDL surface; draw shadow (black, offset 1,1) if styleShadow set, then main text
- Inputs: `s` ΓÇö target surface; `text` ΓÇö string; `length` ΓÇö char count; `x,y` ΓÇö position; `pixel` ΓÇö color; `style` ΓÇö style flags; `utf8` ΓÇö encoding
- Outputs/Return: Width drawn (from `_draw_text()`)
- Side effects: Modifies SDL surface pixel data
- Calls: `_draw_text()` (virtual, implemented in screen_drawing.cpp), `SDL_MapRGB()`
- Notes: Shadow color is black (0x000000); `_draw_text()` not defined in this file

### font_info::draw_styled_text(SDL_Surface *s, const std::string& text, size_t length, int x, int y, uint32 pixel, uint16 style, bool utf8)
- Signature: `int font_info::draw_styled_text(...) const`
- Purpose: Render styled text; tokenize on style codes (|b, |i, |p), update style state, render segments
- Inputs: `s` ΓÇö surface; `text` ΓÇö string with style codes; `length` ΓÇö substring length; `x,y` ΓÇö position; `pixel` ΓÇö color; `style` ΓÇö initial style; `utf8` ΓÇö encoding
- Outputs/Return: Total width drawn
- Side effects: Modifies surface; accumulates width across segments
- Calls: `is_style_token()`, `update_style()`, `_draw_text()`
- Notes: Uses Boost tokenizer with custom `style_separator`; shadow applied per-segment if styleShadow set

### font_info::styled_text_width(const std::string& text, size_t length, uint16 style, bool utf8)
- Signature: `int font_info::styled_text_width(...) const`
- Purpose: Measure styled text; accumulate widths of non-style segments, update style on style codes
- Inputs: `text` ΓÇö string with codes; `length` ΓÇö substring; `style` ΓÇö initial style; `utf8` ΓÇö encoding
- Outputs/Return: Width (+ 1 for shadow if set)
- Calls: `is_style_token()`, `update_style()`, `_text_width()`
- Notes: Returns width + 1 if styleShadow set

### font_info::trunc_styled_text(const std::string& text, int max_width, uint16 style)
- Signature: `int font_info::trunc_styled_text(...) const`
- Purpose: Truncate styled text to fit in max_width; account for style codes (2 chars each)
- Inputs: `text` ΓÇö string with style codes; `max_width` ΓÇö pixel limit; `style` ΓÇö initial style
- Outputs/Return: Number of characters (including style code chars) that fit
- Side effects: Adjusts max_width for shadow; clears styleShadow from working copy
- Calls: `is_style_token()`, `update_style()`, `_trunc_text()`, `_text_width()`
- Notes: Style codes consume length but don't consume pixels; returns early if segment truncates before end

### font_info::style_at(const std::string& text, std::string::const_iterator pos, uint16 style)
- Signature: `std::string font_info::style_at(...) const`
- Purpose: Determine active style at a text position; tokenize up to pos, apply all style codes
- Inputs: `text` ΓÇö full string; `pos` ΓÇö iterator to position; `style` ΓÇö initial style
- Outputs/Return: Style code string ("|b", "|i", or empty) at position
- Calls: `is_style_token()`, `update_style()`
- Notes: Returns "|b" for bold, "|i" for italic, empty string for plain

**Notes:**
- Trivial helper functions: `style_code()` (check char against 'p','b','i','l','r','c','s'), `is_style_token()` (check if string is "|[code]"), `update_style()` (apply token to style flags)

## Control Flow Notes
**Initialization:** `initialize_fonts()` is called at engine startup; searches `data_search_path` for "Fonts" resource file, opens via resource manager. If TTF support available but file not found, uses built-in fallback fonts.

**Font Load:** `load_font()` (public) ΓåÆ TTF path if available (with cascading fallbacks) ΓåÆ bitmap font via `load_sdl_font()` (cache lookup, FONDΓåÆNFNT/FONT resource extraction, bitmap expansion). All fonts cached by ID/size with reference counting.

**Text Rendering:** `draw_text()` ΓåÆ optional shadow draw + main draw via `_draw_text()` (defined in screen_drawing.cpp). Styled text: `draw_styled_text()` tokenizes, applies style codes, renders segments.

**Unloading:** `unload_font()` ΓåÆ virtual `_unload()` ΓåÆ decrement ref count, delete on zero.

## External Dependencies
- **SDL:** `<SDL_endian.h>` (byte-order I/O), SDL_RWops, SDL_ttf (TTF_Font, TTF_GlyphMetrics, TTF_SizeUTF8/UNICODE, TTF_OpenFont, TTF_SetFontStyle, TTF_SetFontHinting, TTF_CloseFont)
- **Boost:** `<boost/tokenizer.hpp>` (custom tokenizer separator)
- **Engine headers:** `cseries.h`, `sdl_fonts.h` (type defs), `byte_swapping.h`, `resource_manager.h` (get_resource, LoadedResource, open_res_file), `FileHandler.h` (FileSpecifier), `Logging.h` (logContext, logFatal), `preferences.h` (environment_preferences), `AlephSansMono-Bold.h` (embedded font data)
- **Defined elsewhere:** `mac_roman_to_unicode()` (encoding conversion), `_draw_text()` methods (screen_drawing.cpp), `fix_missing_*()` functions, `data_search_path` (shell_sdl.cpp)
