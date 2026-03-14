# Source_Files/RenderOther/FontHandler.cpp

## File Purpose
Implements font specification, management, and rendering for the Aleph One game engine. Supports platform-specific font handling (MacOS/SDL) and OpenGL-accelerated text rendering. Manages font parameters, metrics, textures, and provides XML configuration parsing.

## Core Responsibilities
- Font parameter management (name, size, style, file path) with platform-specific initialization
- Text width calculation and character glyph width lookup tables
- OpenGL font texture generation with glyph packing and display list creation
- Platform-specific text rendering (MacOS Quickdraw, SDL surfaces, OpenGL)
- Font name list parsing with fallback/preference support
- Centralized cleanup via font registry for all active OpenGL fonts
- XML configuration parsing for batch font attribute updates

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FontSpecifier | class | Font specification and rendering engine; stores metrics (ascent, descent, leading, widths[256]) and platform-specific state (ID on MacOS, Info* on SDL, OGL textures) |
| XML_FontParser | class | Derives from XML_ElementParser; parses XML font attributes (index, name, size, style, file) into FontSpecifier array |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| m_font_registry | set<FontSpecifier*> | static class member | Tracks all active FontSpecifier instances for coordinated OpenGL cleanup during shutdown |
| FontParser | XML_FontParser | static | Singleton parser reused across multiple XML font configurations |

## Key Functions / Methods

### FontSpecifier::Init
- Signature: `void Init()`
- Purpose: Initialize font specifier before use; clears SDL font info and calls Update()
- Inputs: None
- Outputs/Return: None
- Side effects: Clears OGL_Texture, allocates/resets platform-specific state
- Calls: Update()
- Notes: Must be called before any other font operations; compensates for lack of proper constructor

### FontSpecifier::Update
- Signature: `void Update()` (platform-specific: `#if defined(mac)` / `#elif defined(SDL)`)
- Purpose: Synchronize font metrics (ascent, descent, leading, line spacing) and glyph widths with current font parameters
- Inputs: NameSet, Size, Style (MacOS); File, AdjustLineHeight (SDL)
- Outputs/Return: Populates Ascent, Descent, Leading, Height, LineSpacing, Widths[256]
- Side effects: Calls platform APIs (GetFNum, GetFontInfo on MacOS; load_font on SDL); unloads old font info on SDL
- Calls: GetFont/SetFont (MacOS), Use() (MacOS), load_font/unload_font (SDL)
- Notes: MacOS version tries font names in NameSet order until one is found; SDL version falls back to filename or hardcoded ID

### FontSpecifier::TextWidth
- Signature: `int TextWidth(const char *Text)` (platform-specific)
- Purpose: Calculate total pixel width of a C-string
- Inputs: Text pointer (null-terminated)
- Outputs/Return: Width in pixels
- Side effects: Temporarily sets font (MacOS); reads from Widths[] table (SDL)
- Calls: ::TextWidth() (MacOS), CharWidth() (implicit via Widths[] on SDL)
- Notes: MacOS version caps string length to 255; SDL version sums individual character widths

### FontSpecifier::OGL_Reset
- Signature: `void OGL_Reset(bool IsStarting)`
- Purpose: Create or teardown OpenGL font texture and display lists; allocates LA88 (luminance-alpha) texture from rendered glyphs
- Inputs: IsStarting (true = create, false = cleanup)
- Outputs/Return: None
- Side effects: Allocates/deletes OGL_Texture, TxtrID, DispList; registers/deregisters in m_font_registry; platform-specific glyph rendering (MacOS GWorld or SDL surface)
- Calls: glGenTextures, glBindTexture, glTexImage2D, glGenLists, glNewList, glTranslatef, glBegin/glEnd, glTexCoord2f, glVertex2d (OpenGL); NewGWorld/DisposeGWorld (MacOS); SDL_CreateRGBSurface/SDL_FreeSurface (SDL); draw_text (SDL via sdl_font_info)
- Notes: Pads glyphs by 1 pixel to avoid clipping; packs glyphs left-to-right, wraps to next line; computes power-of-two texture dimensions; creates individual display lists per character for efficient rendering

### FontSpecifier::OGL_Render
- Signature: `void OGL_Render(const char *Text)`
- Purpose: Render a C-string in OpenGL using pre-built display lists
- Inputs: Text pointer (null-terminated, max 255 chars)
- Outputs/Return: None
- Side effects: Modifies modelview matrix; enables texturing, blending; alters GL attributes
- Calls: OGL_Reset(true) (lazy init), glPushAttrib, glEnable/glDisable, glBindTexture, glCallList
- Notes: Assumes screen coordinates and left baseline at (0,0); caller should surround with glPushMatrix/glPopMatrix if preservation needed; lazy-initializes texture on first call

### FontSpecifier::OGL_DrawText
- Signature: `void OGL_DrawText(const char *text, const screen_rectangle &r, short flags)`
- Purpose: Render text with alignment (horizontal/vertical), wrapping, and truncation; handles recursive wrapping for multi-line text
- Inputs: text string, destination rectangle, alignment flags (_center_horizontal, _center_vertical, _right_justified, _top_justified, _wrap_text)
- Outputs/Return: None
- Side effects: Modifies modelview matrix; recursive calls for wrapped lines
- Calls: TextWidth, CharWidth, OGL_Render, memcpy (string handling)
- Notes: Wrapping breaks at last space within rectangle width; vertical centering disabled if wrapping occurs; truncates text if it exceeds rectangle width

### FontSpecifier::OGL_ResetFonts (static)
- Signature: `static void OGL_ResetFonts(bool IsStarting)`
- Purpose: Cleanup all registered fonts; iterates m_font_registry and calls OGL_Reset on each
- Inputs: IsStarting (false for cleanup)
- Outputs/Return: None
- Side effects: Global cleanup of all font textures and display lists
- Calls: OGL_Reset(false) on each registered FontSpecifier
- Notes: Early return if IsStarting=true (no action on init); safe iteration by always fetching begin() since OGL_Reset removes elements

### FontSpecifier::FindNextName (static)
- Signature: `static char *FindNextName(char *NamePtr)`
- Purpose: Skip delimiters and return pointer to next font name in comma/semicolon-separated list
- Inputs: Pointer anywhere in font name string
- Outputs/Return: Pointer to first non-delimiter character, or NULL if end of string
- Side effects: None
- Calls: None
- Notes: Skips whitespace, commas, semicolons, carriage returns

### FontSpecifier::FindNameEnd (static)
- Signature: `static char *FindNameEnd(char *NamePtr)`
- Purpose: Find end of current font name (delimiter or null terminator)
- Inputs: Pointer to start of font name
- Outputs/Return: Pointer to delimiter or null terminator
- Side effects: None
- Calls: None
- Notes: Stops at comma or semicolon

### XML_FontParser::HandleAttribute
- Signature: `bool HandleAttribute(const char *Tag, const char *Value)`
- Purpose: Parse single XML font attribute (index, name, size, style, file)
- Inputs: XML tag name and string value
- Outputs/Return: true on success, false on error
- Side effects: Populates TempFont or sets IsPresent flags
- Calls: ReadBoundedInt16Value, ReadInt16Value, strncpy, StringsEqual, UnrecognizedTag
- Notes: Only allows index attribute if NumFonts > 0; defaults to index=0 if no NumFonts; validates bounds and format

### XML_FontParser::AttributesDone
- Signature: `bool AttributesDone()`
- Purpose: Validate parsed attributes and commit to FontList[Index]; triggers Update() if any parameter changed
- Inputs: None (reads IsPresent[], TempFont, FontList, Index, NumFonts)
- Outputs/Return: true if successful and committed, false if validation failed
- Side effects: Modifies FontList[Index] and calls Update() if attributes changed
- Calls: Update() on modified font
- Notes: Requires index attribute if NumFonts > 0; copies only present attributes to avoid overwriting defaults

## Control Flow Notes
**Initialization:** `Init()` ΓåÆ `Update()` to populate metrics; `OGL_Reset(true)` creates textures on first render.

**Frame/Render:** `OGL_DrawText()` (high-level) or `OGL_Render()` (low-level) renders glyphs via pre-built display lists.

**Shutdown:** `OGL_ResetFonts(false)` walks registry and calls `OGL_Reset(false)` per font, deallocating GPU memory.

**Configuration:** XML parser invokes `HandleAttribute()` per tag, then `AttributesDone()` to validate and commit changes; triggers `Update()` to recompute metrics if needed.

## External Dependencies
- **OpenGL:** glGenTextures, glBindTexture, glTexImage2D, glGenLists, glNewList, glCallList, glPushMatrix, glPopMatrix, glTranslatef/d, glBegin/glEnd, glTexCoord2f, glVertex2d, glPushAttrib, glPopAttrib, glEnable/glDisable
- **MacOS:** TextSpec, TextFont, TextFace, TextSize, GetFontInfo, GetFNum, NewGWorld, GetGWorldPixMap, LockPixels, GetGWorld, SetGWorld, BackColor, ForeColor, EraseRect, MoveTo, DrawChar, DisposeGWorld, Rect, GWorldPtr, PixMapHandle, GetPixBaseAddr
- **SDL:** SDL_Surface, SDL_CreateRGBSurface, SDL_FillRect, SDL_MapRGB, SDL_FreeSurface; load_font, unload_font, draw_text, char_width (from sdl_fonts.h / screen_drawing.cpp)
- **Game engine:** screen_rectangle (screen_drawing.h), XML_ElementParser (base class), shape_descriptors.h (included but unused here)
