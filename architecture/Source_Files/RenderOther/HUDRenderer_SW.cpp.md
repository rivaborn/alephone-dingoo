# Source_Files/RenderOther/HUDRenderer_SW.cpp

## File Purpose
Software-rendered HUD implementation for the Aleph One game engine using SDL surfaces. Provides concrete implementations of HUD rendering methods including motion sensor updates, shape and texture drawing, text rendering, and basic rasterization primitives (rectangles).

## Core Responsibilities
- Update and render the motion sensor display with state change detection
- Draw shapes from the shape collection at specified screen coordinates
- Render textured shapes with automatic aspect-ratio-preserving scaling using Shape_Blitter
- Draw screen text with font and color control
- Rasterize filled and outlined rectangles
- Provide SDL surface rotation utility for pixel data transformation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef | Encodes collection, CLUT, and shape index for shape resources |
| `screen_rectangle` | struct | Rectangular region on screen (left, top, right, bottom) |
| `point2d` | struct | 2D coordinate pair for entity locations |
| `SDL_Surface` | struct (SDL) | Pixel buffer with format metadata |
| `SDL_Rect` | struct (SDL) | Integer rectangle for blit operations |
| `Shape_Blitter` | class | Handles shape rendering with scaling and tinting |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `HUD_Buffer` | `SDL_Surface*` | extern | Target surface for all HUD drawing operations |
| `MotionSensorActive` | `bool` | extern | Controls whether motion sensor updates are performed |

## Key Functions / Methods

### `update_motion_sensor`
- **Signature:** `void HUD_SW_Class::update_motion_sensor(short time_elapsed)`
- **Purpose:** Update motion sensor state if enabled and active; redraw if changed
- **Inputs:** `time_elapsed` ΓÇô milliseconds since last update (NONE = full reset)
- **Outputs/Return:** None (modifies global state)
- **Side effects:** Calls `reset_motion_sensor()`, `motion_sensor_scan()`, sets `ForceUpdate` flag, draws motion sensor shape to screen
- **Calls:** `reset_motion_sensor()`, `motion_sensor_scan()`, `motion_sensor_has_changed()`, `get_interface_rectangle()`, `DrawShapeAtXY()`
- **Notes:** Guarded by `_motion_sensor_does_not_work` game option and `MotionSensorActive` flag

### `DrawShape`
- **Signature:** `void HUD_SW_Class::DrawShape(shape_descriptor shape, screen_rectangle *dest, screen_rectangle *src)`
- **Purpose:** Draw a shape sprite with source and destination rectangles
- **Inputs:** `shape` ΓÇô shape resource descriptor; `dest` ΓÇô screen destination region; `src` ΓÇô source region within shape
- **Outputs/Return:** None
- **Side effects:** Writes to HUD_Buffer via `_draw_screen_shape()`
- **Calls:** `_draw_screen_shape()` (defined elsewhere)
- **Notes:** Delegates to lower-level software rasterizer

### `DrawShapeAtXY`
- **Signature:** `void HUD_SW_Class::DrawShapeAtXY(shape_descriptor shape, short x, short y, bool transparency = false)`
- **Purpose:** Draw a shape at integer coordinates
- **Inputs:** `shape` ΓÇô shape descriptor; `x`, `y` ΓÇô screen coordinates; `transparency` ΓÇô unused in software renderer
- **Outputs/Return:** None
- **Side effects:** Writes to HUD_Buffer
- **Calls:** `_draw_screen_shape_at_x_y()` (defined elsewhere)
- **Notes:** `transparency` parameter exists for OpenGL compatibility only

### `DrawTexture`
- **Signature:** `void HUD_SW_Class::DrawTexture(shape_descriptor shape, short texture_type, short x, short y, int size)`
- **Purpose:** Draw a shape as a textured quad, auto-scaled to fit a square with aspect ratio preservation
- **Inputs:** `shape` ΓÇô shape descriptor; `texture_type` ΓÇô one of Shape_Texture_* enums; `x`, `y` ΓÇô top-left corner; `size` ΓÇô target square dimension
- **Outputs/Return:** None
- **Side effects:** Creates Shape_Blitter, rescales, and blits to HUD_Buffer
- **Calls:** `Shape_Blitter` constructor, `Rescale()`, `Width()`, `Height()`, `SDL_Draw()`
- **Notes:** Maintains aspect ratio; centers scaled result within the size├ùsize region

### `DrawText`
- **Signature:** `void HUD_SW_Class::DrawText(const char *text, screen_rectangle *dest, short flags, short font_id, short text_color)`
- **Purpose:** Render text to screen
- **Inputs:** `text` ΓÇô null-terminated string; `dest` ΓÇô destination rectangle; `flags` ΓÇô text alignment/style flags; `font_id` ΓÇô font selection; `text_color` ΓÇô color index
- **Outputs/Return:** None
- **Side effects:** Writes to HUD_Buffer
- **Calls:** `_draw_screen_text()` (defined elsewhere)

### `FillRect`
- **Signature:** `void HUD_SW_Class::FillRect(screen_rectangle *r, short color_index)`
- **Purpose:** Fill a rectangle with a solid color
- **Inputs:** `r` ΓÇô rectangle to fill; `color_index` ΓÇô palette or color reference
- **Outputs/Return:** None
- **Side effects:** Writes to HUD_Buffer
- **Calls:** `_fill_rect()` (defined elsewhere)

### `FrameRect`
- **Signature:** `void HUD_SW_Class::FrameRect(screen_rectangle *r, short color_index)`
- **Purpose:** Draw an outlined rectangle (frame)
- **Inputs:** `r` ΓÇô rectangle to frame; `color_index` ΓÇô outline color
- **Outputs/Return:** None
- **Side effects:** Writes to HUD_Buffer
- **Calls:** `_frame_rect()` (defined elsewhere)

### `rotate_surface` (utility)
- **Signature:** `SDL_Surface *rotate_surface(SDL_Surface *s, int width, int height)`
- **Purpose:** Rotate an SDL surface 90┬░ clockwise by swapping x/y coordinates
- **Inputs:** `s` ΓÇô source surface; `width`, `height` ΓÇô original dimensions
- **Outputs/Return:** New rotated SDL_Surface (or nullptr if input is null)
- **Side effects:** Allocates new SDL_Surface; handles 1, 2, 4-byte-per-pixel formats; copies palette if present
- **Calls:** `SDL_CreateRGBSurface()`, `rotate<T>()` template, `SDL_SetColors()`
- **Notes:** Templated rotate helper dispatches on `BytesPerPixel`; caller must free returned surface

## Control Flow Notes
- This class implements the software-renderer half of the HUD subsystem; inherits virtual methods from `HUD_Class`
- **Frame phase:** `update_motion_sensor()` is called once per frame to update sensor state
- **Render phase:** `DrawShape*()`, `DrawTexture()`, `DrawText()`, `Fill/FrameRect()` are called to composite HUD elements onto `HUD_Buffer`
- All drawing operations delegate to screen_drawing module functions (prefixed `_`) that perform the actual rasterization

## External Dependencies
- **SDL:** `SDL_Surface`, `SDL_Rect`, `SDL_CreateRGBSurface()`, `SDL_SetColors()`
- **images.h:** Image/resource management (implied by included files)
- **shell.h:** `get_shape_surface()` (commented in include as "get_shape_surface!?")
- **Shape_Blitter.h:** `Shape_Blitter` class for textured shape rendering
- **Defined elsewhere:** `_draw_screen_shape()`, `_draw_screen_shape_at_x_y()`, `_draw_screen_text()`, `_fill_rect()`, `_frame_rect()`, `reset_motion_sensor()`, `motion_sensor_scan()`, `motion_sensor_has_changed()`, `get_interface_rectangle()`, `GET_GAME_OPTIONS()`, `GET_DESCRIPTOR_*()` macros, `BUILD_DESCRIPTOR()` macro
- **Global state:** `HUD_Buffer` (extern SDL_Surface*), `MotionSensorActive` (extern bool), `current_player_index` (implied from `reset_motion_sensor()` call), `ForceUpdate` (class member flag)
