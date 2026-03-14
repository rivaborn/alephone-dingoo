# Source_Files/RenderOther/FontHandler.h

## File Purpose
Defines the FontSpecifier class for managing font specifications, metrics, and rendering across MacOS/SDL/OpenGL platforms. Handles font parameter specification, metric derivation, and multi-platform text rendering including OpenGL texture-based font rendering.

## Core Responsibilities
- Store and manage font parameters (name fallback list, size, style, line height adjustment)
- Compute and cache derived font metrics (height, line spacing, ascent/descent/leading, per-character widths)
- Provide text width queries for layout operations (centering, truncation)
- Render text in OpenGL using pre-built font textures and display lists
- Support platform-specific font systems (MacOS QuickDraw, SDL, OpenGL)
- Parse and update font specifications from XML configuration
- Manage global font registry for OpenGL resource cleanup

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FontSpecifier | class | Main font management class; stores parameters, metrics, and platform-specific data |
| font_info | class (from sdl_fonts.h) | Abstract base for SDL font information and rendering |
| screen_rectangle | struct (forward declared) | Rectangle for text layout bounds and alignment |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| m_font_registry | static set<FontSpecifier*> | static (class member) | Registry of all active FontSpecifier instances for batch OpenGL cleanup |

## Key Functions / Methods

### Init
- **Signature:** `void Init()`
- **Purpose:** Initialize FontSpecifier before use (C-style constructor alternative)
- **Inputs:** None (uses member fields)
- **Outputs/Return:** None
- **Side effects:** Populates Height, LineSpacing, Ascent, Descent, Leading, Widths from font system
- **Calls:** Update()
- **Notes:** Must be called before any other methods; invoked once at object construction

### Update
- **Signature:** `void Update()`
- **Purpose:** Recompute all derived font metrics from parameter fields
- **Inputs:** NameSet, Size, Style, AdjustLineHeight
- **Outputs/Return:** None
- **Side effects:** Updates Height, LineSpacing, Ascent, Descent, Leading, Widths[256]
- **Calls:** Platform-specific font metric queries
- **Notes:** Called after Init() and whenever XML parser modifies parameters

### TextWidth
- **Signature:** `int TextWidth(const char *Text)`
- **Purpose:** Get pixel width of text string for layout operations
- **Inputs:** C-string text
- **Outputs/Return:** Width in pixels
- **Side effects:** None
- **Calls:** CharWidth() for each character
- **Notes:** Used for centering operations (map titles)

### OGL_Render
- **Signature:** `void OGL_Render(const char *Text)`
- **Purpose:** Render text string in OpenGL using screen coordinates
- **Inputs:** C-string text
- **Outputs/Return:** None
- **Side effects:** Modifies OpenGL modelview matrix; assumes baseline at (0,0)
- **Calls:** OpenGL drawing commands via display list
- **Notes:** Can be wrapped with glPushMatrix/glPopMatrix to preserve matrix state; requires OGL_Reset() first

### OGL_DrawText
- **Signature:** `void OGL_DrawText(const char *Text, const screen_rectangle &r, short flags)`
- **Purpose:** Render text with alignment and wrapping (like screen_drawing.h's _draw_screen_text)
- **Inputs:** Text, bounding rectangle, alignment flags
- **Outputs/Return:** None
- **Side effects:** None (modelview matrix unaffected)
- **Calls:** OpenGL rendering commands
- **Notes:** Handles text alignment and line wrapping within bounds

### OGL_Reset
- **Signature:** `void OGL_Reset(bool IsStarting)`
- **Purpose:** Manage OpenGL texture and display list resources for this font
- **Inputs:** IsStarting (true = session initialization, false = mid-session)
- **Outputs/Return:** None
- **Side effects:** Allocates/deallocates OGL_Texture and display lists; prevents memory leaks
- **Calls:** OpenGL texture/display list APIs
- **Notes:** IsStarting flag avoids leaks when resetting during session

### OGL_ResetFonts
- **Signature:** `static void OGL_ResetFonts(bool IsStarting)`
- **Purpose:** Reset all registered fonts (batch operation)
- **Inputs:** IsStarting flag
- **Outputs/Return:** None
- **Side effects:** Calls OGL_Reset on all entries in m_font_registry
- **Calls:** OGL_Reset() for each registered font
- **Notes:** Static utility for global cleanup

### FindNextName / FindNameEnd
- **Signature:** `static char *FindNextName(char *NamePtr)` / `static char *FindNameEnd(char *NamePtr)`
- **Purpose:** Parse HTML-style comma-separated font name list ("font1, font2, font3")
- **Inputs:** Pointer into NameSet string
- **Outputs/Return:** Pointer to next name start (FindNextName) or character after current name (FindNameEnd); NULL if no more names
- **Side effects:** None
- **Calls:** None
- **Notes:** Used together to iterate through fallback font list

**Trivial helpers:** CharWidth() ΓÇö inline lookup into Widths array; Use() ΓÇö platform-specific font activation (MacOS only); equality operators (==, !=, =)

## Control Flow Notes
- **Initialization:** Init() ΓåÆ Update() ΓåÆ platform-specific font loading
- **XML updates:** Parser calls Update() after modifying parameters
- **OpenGL rendering:** OGL_Reset(true) at session start ΓåÆ OGL_Render/OGL_DrawText during frame ΓåÆ OGL_Reset(false) at cleanup
- **MacOS rendering:** Use() sets global QuickDraw font, then native drawing functions follow
- **Font discovery:** NameSet holds fallback list; Update() uses first available font via FindNextName/FindNameEnd

## External Dependencies
- **cseries.h:** Core types, platform macros (mac, SDL, HAVE_OPENGL), Str255, struct Rect
- **XML_ElementParser.h:** Font_GetParser(), Font_SetArray() for XML-driven font configuration
- **sdl_fonts.h:** font_info class (when SDL defined)
- **GL/gl.h:** OpenGL types/functions (GLuint, glPushMatrix, etc.) when HAVE_OPENGL defined
- **\<set\>:** STL container for m_font_registry
