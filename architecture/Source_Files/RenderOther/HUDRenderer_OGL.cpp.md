# Source_Files/RenderOther/HUDRenderer_OGL.cpp

## File Purpose
OpenGL-based HUD renderer for the Aleph One game engine. Responsible for rendering the player-facing heads-up display showing weapons, ammo, shields, oxygen, inventory, motion sensor, and messages each frame.

## Core Responsibilities
- Load and draw the static HUD backdrop panel
- Render dynamic HUD elements (weapon/ammo/shield/oxygen displays, inventory)
- Draw shaped interface graphics with texture support and scaling
- Render text labels and messages with specified fonts and colors
- Fill and frame rectangles for UI elements
- Manage motion sensor display with circular clipping regions
- Set up OpenGL state and viewport for HUD rendering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `HUD_OGL_Class` | class | Main HUD renderer; inherits from `HUD_Class` and implements OpenGL drawing methods |
| `TextureManager` | class (imported) | Manages OpenGL texture state, shading tables, and coordinate mapping for shape rendering |
| `OGL_Blitter` | class (imported) | Loads and draws the static HUD backdrop image |
| `Shape_Blitter` | class (imported) | Renders shapes with scaling and repositioning |
| `FontSpecifier` | class (imported) | Manages font properties and OpenGL text rendering |
| `screen_rectangle` | struct (imported) | Screen-space rectangle with left, top, right, bottom coordinates |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `HUD_OGL` | `HUD_OGL_Class` | static | Singleton HUD renderer instance used throughout frame |
| `HUD_Blitter` | `OGL_Blitter` | static | Cached HUD backdrop texture; loaded once, reused each frame |
| `hud_pict_not_found` | `bool` | static | Flag to prevent repeated failed load attempts for HUD backdrop |
| `MotionSensorActive` | `bool` | extern | Game state indicating motion sensor is available/active |
| `current_player_index` | int | extern | Currently active player index for HUD updates |

## Key Functions / Methods

### OGL_DrawHUD
- **Signature:** `void OGL_DrawHUD(Rect &dest, short time_elapsed)`
- **Purpose:** Main entry point for rendering the HUD each frame; sets up OpenGL state, draws static backdrop, marks elements dirty, and updates all dynamic content
- **Inputs:** `dest` (HUD viewport rectangle), `time_elapsed` (frame delta for animation)
- **Outputs/Return:** None (side effects to OpenGL framebuffer)
- **Side effects:** Modifies OpenGL state (texture mode, blend, depth test, scissor, matrix); calls marking functions; invokes `update_everything()`
- **Calls:** `HUD_Blitter.Load()`, `HUD_Blitter.Draw()`, `glPushAttrib/glPopAttrib`, `glScissor`, `glTranslated`, `glScaled`, marking functions, `HUD_OGL.update_everything()`
- **Notes:** Lumatex palette presence disables texture rendering; scissor region constrains drawing to HUD area; modelview matrix is transformed to HUD-local coordinates

### HUD_OGL_Class::DrawShape
- **Signature:** `void DrawShape(shape_descriptor shape, screen_rectangle *dest, screen_rectangle *src)`
- **Purpose:** Render a shape descriptor (sprite/bitmap) into a screen rectangle with optional source clipping
- **Inputs:** `shape` (identifier for shape/frame), `dest` (target rectangle), `src` (source clipping region)
- **Outputs/Return:** None
- **Side effects:** Modifies texture, matrix, and OpenGL state; binds texture; renders quad
- **Calls:** `get_shape_bitmap_and_shading_table()`, `TextureManager::Setup()`, `TextureManager::SetupTextureMatrix()`, `TextureManager::RenderNormal()`, `glBegin/glEnd`
- **Notes:** Early return if `TextureManager::Setup()` fails; UV coordinates computed from source region

### HUD_OGL_Class::DrawShapeAtXY
- **Signature:** `void DrawShapeAtXY(shape_descriptor shape, short x, short y, bool transparency = false)`
- **Purpose:** Render a shape at absolute screen coordinates with optional alpha blending
- **Inputs:** `shape` (shape descriptor), `x`, `y` (screen position), `transparency` (blend mode flag)
- **Outputs/Return:** None
- **Side effects:** Binds texture; sets blend function if transparency enabled; renders textured quad
- **Calls:** `TextureManager` setup and rendering as above; `glBlendFunc`
- **Notes:** Preserves texture dimensions; uses full texture UV range

### HUD_OGL_Class::DrawTexture
- **Signature:** `void DrawTexture(shape_descriptor shape, short texture_type, short x, short y, int size)`
- **Purpose:** Render a shape as a square icon with uniform scaling into a bounding box
- **Inputs:** `shape` (shape descriptor), `texture_type` (type constant), `x`, `y` (box position), `size` (box extent)
- **Outputs/Return:** None
- **Side effects:** Creates Shape_Blitter; rescales to fit box aspect; renders via `OGL_Draw()`
- **Calls:** `Shape_Blitter` constructor, `Rescale()`, `OGL_Draw()`
- **Notes:** Maintains aspect ratio; centers sprite within bounding box

### HUD_OGL_Class::DrawText
- **Signature:** `void DrawText(const char *text, screen_rectangle *dest, short flags, short font_id, short text_color)`
- **Purpose:** Render a text string with color, font, and alignment/wrapping
- **Inputs:** `text` (C string), `dest` (bounding rectangle), `flags` (alignment/wrapping), `font_id` (font index), `text_color` (color index)
- **Outputs/Return:** None
- **Side effects:** Sets OpenGL color; retrieves and uses font specifier
- **Calls:** `get_interface_color()`, `get_interface_font()`, `FontSpecifier::OGL_DrawText()`, `SglColor3us()`
- **Notes:** Color and font resolved from interface palette/settings

### HUD_OGL_Class::FillRect / FrameRect
- **Signature:** `void FillRect(screen_rectangle *r, short color_index)` / `void FrameRect(screen_rectangle *r, short color_index)`
- **Purpose:** Fill or outline a rectangle with a color
- **Inputs:** `r` (rectangle), `color_index` (palette index)
- **Outputs/Return:** None
- **Side effects:** Disables texture; sets color; renders via `glRecti` (fill) or `glLineLoop` (outline)
- **Calls:** `get_interface_color()`, `SglColor3us()`, OpenGL drawing commands
- **Notes:** FrameRect uses 0.5 pixel offsets to align lines with pixel centers

### HUD_OGL_Class::SetClipPlane / DisableClipPlane
- **Signature:** `void SetClipPlane(int x, int y, int c_x, int c_y, int radius)` / `void DisableClipPlane(void)`
- **Purpose:** Enable circular clipping for motion sensor blips; plane tangent to circle at blip location
- **Inputs:** `x`, `y` (blip offset from center), `c_x`, `c_y` (circle center), `radius` (circle radius)
- **Outputs/Return:** None
- **Side effects:** Enables/disables `GL_CLIP_PLANE0`; sets plane equation
- **Calls:** `glEnable`, `glDisable`, `glClipPlane`, `sqrt`
- **Notes:** Early return if blip distance < 2.0; plane normal computed from blip direction

### HUD_OGL_Class::update_motion_sensor
- **Signature:** `void update_motion_sensor(short time_elapsed)`
- **Purpose:** Update motion sensor state and graphics if active
- **Inputs:** `time_elapsed` (frame delta; NONE resets sensor)
- **Outputs/Return:** None
- **Side effects:** Calls motion sensor update functions
- **Calls:** `reset_motion_sensor()`, `motion_sensor_scan()`
- **Notes:** Conditional on game options and `MotionSensorActive` flag

### HUD_OGL_Class::draw_message_area
- **Signature:** `void draw_message_area(short)`
- **Purpose:** Render player name display area with network panel background
- **Inputs:** Unused parameter
- **Outputs/Return:** None
- **Side effects:** Draws panel background and player name
- **Calls:** `get_interface_rectangle()`, `DrawShapeAtXY()`, `draw_player_name()`
- **Notes:** Message area is offset by defined constants (`MESSAGE_AREA_X_OFFSET`, `MESSAGE_AREA_Y_OFFSET`)

## Control Flow Notes
`OGL_DrawHUD()` is the frame-level entry point, called during the render phase. It initializes OpenGL state, renders the static HUD panel, sets up a scissored viewport and transformed modelview matrix (translating and scaling to 640├ù160 HUD coordinates), marks all dynamic element displays dirty, and calls `HUD_OGL.update_everything()` (inherited from `HUD_Class`) to render weapons, ammo, shields, oxygen, inventory, and motion sensor. The motion sensor update is conditional on game options.

## External Dependencies
- **OpenGL:** `GL/gl.h` or `OpenGL/gl.h` (platform-conditional)
- **FontHandler:** `FontSpecifier` for text rendering; `get_interface_font()`
- **game_window:** Functions to mark HUD elements dirty (`mark_*_display_as_dirty()`)
- **OGL infrastructure:** `OGL_Setup.h`, `OGL_Textures.h`, `OGL_Blitter`, `Shape_Blitter`, `TextureManager`
- **Images/Resources:** `get_shape_bitmap_and_shading_table()`, `INTERFACE_PANEL_BASE` constant
- **Interface palette:** `get_interface_color()`, `get_interface_rectangle()`
- **Lua:** `LuaTexturePaletteSize()` for dynamic palette override
- **Render state:** `_shading_normal`, `_shadeless_transfer`, `OGL_Txtr_WeaponsInHand`
- **Imported types:** `shape_descriptor`, `screen_rectangle`, `Rect`, `SDL_Rect`, `rgb_color`
