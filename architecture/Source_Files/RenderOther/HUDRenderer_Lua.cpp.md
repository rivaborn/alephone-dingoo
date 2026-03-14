# Source_Files/RenderOther/HUDRenderer_Lua.cpp

## File Purpose
Implements Lua-scripted HUD rendering for Aleph One. Provides a C++ bridge layer that exposes drawing primitives (rectangles, text, images, shapes) to Lua scripts, with dual OpenGL and SDL backend support.

## Core Responsibilities
- **Lifecycle management**: Initialize and finalize drawing state for each frame (`start_draw()`, `end_draw()`)
- **Motion sensor**: Update and manage entity detection blips from the motion sensor
- **Clip regions**: Apply scissor/clip rectangles for constrained drawing areas
- **Primitive rendering**: Draw filled/outlined rectangles, text, images, and shapes in both OpenGL and SDL
- **Backend abstraction**: Provide unified drawing API with conditional OpenGL or SDL implementation
- **Entity blips**: Maintain a list of motion-sensor blips (radar contacts) that Lua can query

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `blip_info` | struct | Motion sensor entity: type, intensity, distance, direction |
| `HUD_Lua_Class` | class | Main HUD renderer; inherits from `HUD_Class` |
| `FontSpecifier` | class | Font specification and rendering (imported) |
| `Image_Blitter` | class | Image drawing primitive (imported) |
| `Shape_Blitter` | class | Shape/bitmap drawing primitive (imported) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `HUD_Lua` | `HUD_Lua_Class` | static | Singleton HUD renderer instance |
| `MotionSensorActive` | bool | extern | Game state flag indicating whether motion sensor is active |

## Key Functions / Methods

### Lua_HUDInstance
- **Signature:** `HUD_Lua_Class *Lua_HUDInstance()`
- **Purpose:** Accessor for the global HUD_Lua singleton
- **Inputs:** None
- **Outputs/Return:** Pointer to static `HUD_Lua` instance
- **Side effects:** None
- **Calls:** None visible
- **Notes:** Provides single entry point to HUD renderer from Lua script system

### Lua_DrawHUD
- **Signature:** `void Lua_DrawHUD(short time_elapsed)`
- **Purpose:** Main frame entry point; orchestrates HUD rendering for one frame
- **Inputs:** `time_elapsed` ΓÇô elapsed milliseconds since last frame (or `NONE` to reset)
- **Outputs/Return:** None
- **Side effects:** Updates motion sensor, modifies OpenGL state, calls Lua script
- **Calls:** `update_motion_sensor()`, `start_draw()`, `L_Call_HUDDraw()`, `end_draw()`
- **Notes:** Called once per frame; coordinates lifecycle

### update_motion_sensor
- **Signature:** `void HUD_Lua_Class::update_motion_sensor(short time_elapsed)`
- **Purpose:** Scan and update motion sensor blips if sensor is active
- **Inputs:** `time_elapsed` ΓÇô elapsed time; `NONE` triggers reset
- **Outputs/Return:** None
- **Side effects:** Calls `motion_sensor_scan()` and `reset_motion_sensor()` (defined elsewhere)
- **Calls:** `motion_sensor_scan()`, `reset_motion_sensor()`, `GET_GAME_OPTIONS()`
- **Notes:** Guarded by game options flag and `MotionSensorActive`

### clear_entity_blips / add_entity_blip / entity_blip_count / entity_blip
- **Purpose:** Manage in-memory blip list (`m_blips` vector)
- **Signature:**
  - `void clear_entity_blips()`
  - `void add_entity_blip(short mtype, short intensity, short x, short y)`
  - `size_t entity_blip_count()`
  - `blip_info entity_blip(size_t index)`
- **Notes:** `add_entity_blip()` computes distance and direction from (0,0) to (x,y); exposed to Lua

### start_draw / end_draw
- **Purpose:** Set up and tear down rendering state for one frame
- **Signature:** `void start_draw()`, `void end_draw()`
- **Side effects (start_draw):**
  - OpenGL path: pushes matrix, sets up 2D projection, disables depth/culling/alpha test
  - SDL path: creates/validates off-screen surface, clears to transparent
  - Sets `m_drawing = true`
- **Side effects (end_draw):**
  - OpenGL: restores matrix and attributes
  - SDL: restores alpha blending
  - Sets `m_drawing = false`
- **Calls:** `alephone::Screen::instance()`, OpenGL calls, SDL calls

### apply_clip
- **Signature:** `void apply_clip()`
- **Purpose:** Apply clip rectangle from screen (bounds drawing)
- **Side effects:**
  - OpenGL: enables `GL_SCISSOR_TEST`, sets scissor rectangle
  - SDL: calls `SDL_SetClipRect()`
- **Calls:** `alephone::Screen::instance()`
- **Notes:** Uses `lua_clip_rect` from Screen singleton

### fill_rect
- **Signature:** `void fill_rect(float x, float y, float w, float h, float r, float g, float b, float a)`
- **Purpose:** Draw a solid colored rectangle
- **Inputs:** Position (x, y), size (w, h), color (RGBA, 0ΓÇô1 range)
- **Side effects:**
  - OpenGL: disables texture, emits quad vertices, restores sRGB state
  - SDL: creates SDL_Rect, fills with mapped RGBA color, blits to video surface
- **Calls:** `apply_clip()`, OpenGL calls or SDL calls

### frame_rect
- **Signature:** `void frame_rect(float x, float y, float w, float h, float r, float g, float b, float a, float t)`
- **Purpose:** Draw a rectangle outline (frame) with thickness `t`
- **Inputs:** Position, size, color (RGBA, 0ΓÇô1 range), thickness `t`
- **Side effects:** Similar to `fill_rect`; OpenGL emits 4 quads for top, bottom, left, right edges
- **Calls:** `apply_clip()`, OpenGL or SDL drawing primitives

### draw_text
- **Signature:** `void draw_text(FontSpecifier *font, const char *text, float x, float y, float r, float g, float b, float a)`
- **Purpose:** Render text string at position with color
- **Inputs:** Font object, null-terminated C string, position, color (RGBA, 0ΓÇô1 range)
- **Side effects:**
  - OpenGL: translates matrix, calls `font->OGL_Render()`
  - SDL: blits background, calls `font->Info->draw_text()`, blits result to video surface
- **Calls:** `apply_clip()`, `font->OGL_Render()` or `font->Info->draw_text()`

### draw_image / draw_shape
- **Signature:**
  - `void draw_image(Image_Blitter *image, float x, float y)`
  - `void draw_shape(Shape_Blitter *shape, float x, float y)`
- **Purpose:** Render preloaded image or shape bitmap at position
- **Inputs:** Image/shape object, position (x, y)
- **Side effects:** Calls `image->Draw()` or `shape->OGL_Draw()`/`shape->SDL_Draw()`
- **Calls:** `apply_clip()`, image/shape draw methods

## Control Flow Notes
**Frame sequence:**
1. `Lua_DrawHUD(elapsed)` called once per frame
2. Motion sensor updated (blip list repopulated)
3. `start_draw()` initializes OpenGL/SDL state
4. `L_Call_HUDDraw()` invokes Lua script, which calls `fill_rect()`, `draw_text()`, etc. via C bindings
5. `end_draw()` cleans up state
6. Control returns; frame presented

The class acts as a drawing backend for a Lua HUD scripting system, supporting both 3D-accelerated (OpenGL) and software (SDL) rendering paths.

## External Dependencies
- **FontHandler.h**: `FontSpecifier` (font metrics, rendering)
- **Image_Blitter.h**: `Image_Blitter` (image drawing)
- **Shape_Blitter.h**: `Shape_Blitter` (shape/bitmap drawing)
- **lua_hud_script.h**: `L_Call_HUDDraw()` (Lua script entry point)
- **shell.h**: `GET_GAME_OPTIONS()`, `MotionSensorActive` (game state)
- **screen.h**: `alephone::Screen::instance()` (window/clip rects)
- **OGL_Setup.h**: `Using_sRGB` flag, OpenGL color setup
- **SDL**: `SDL_Surface`, `SDL_Rect`, `SDL_FillRect()`, `SDL_BlitSurface()`, video surface access
- **OpenGL** (conditional): `glPushAttrib()`, `glVertex2f()`, `glScissor()`, etc.
- **math.h**: `arctangent()` (used in `add_entity_blip()`)
- Defined elsewhere: `motion_sensor_scan()`, `reset_motion_sensor()`, `current_player_index`
