# Source_Files/RenderOther/HUDRenderer_Lua.h

## File Purpose
Defines a Lua-integrated HUD renderer class (`HUD_Lua_Class`) that extends the base `HUD_Class` to provide Aleph One game engine HUD rendering via Lua scripts. Manages motion sensor blips, draws geometric primitives (rectangles, text, images, shapes), and mediates between Lua themes and the underlying rendering backend. Conditionally compiled when Lua support is enabled (`HAVE_LUA`).

## Core Responsibilities
- **Blip management**: Track, update, and query motion sensor entity blips with spatial/intensity data
- **Drawing surface**: Maintain OpenGL/SDL rendering surface state and clipping regions
- **Lua drawing API**: Expose filling/framing rectangles, rendering text/images/shapes at arbitrary coordinates
- **Motion sensor rendering**: Update and draw motion sensor display each frame
- **Base class overrides**: Implement pure virtual drawing methods from `HUD_Class` (mostly as empty stubs for Lua compatibility)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `blip_info` | struct | Motion sensor entity blip: type, intensity, distance, direction |
| `HUD_Lua_Class` | class | Main Lua HUD renderer; extends `HUD_Class` with Lua-friendly public drawing API |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_blips` | `std::vector<blip_info>` | member (protected) | Stores active motion sensor blips |
| `m_drawing` | `bool` | member (protected) | Flag indicating whether drawing pass is active |
| `m_opengl` | `bool` | member (protected) | Flag indicating OpenGL backend in use |
| `m_surface` | `SDL_Surface*` | member (protected) | SDL rendering surface (if software rendering) |
| `m_wr` | `SDL_Rect` | member (protected) | Clipping/window rect for drawing |

## Key Functions / Methods

### `HUD_Lua_Class()` / `~HUD_Lua_Class()`
- Signature: constructors/destructors
- Purpose: Initialize/destroy HUD Lua instance; set `m_drawing = false`
- Side effects: None explicit (trivial)

### `update_motion_sensor(short time_elapsed)`
- Signature: `void update_motion_sensor(short time_elapsed)`
- Purpose: Update motion sensor state (blip decay, motion calculations) each frame
- Inputs: Elapsed time in ticks
- Side effects: Modifies `m_blips` vector

### `clear_entity_blips()` / `add_entity_blip(short mtype, short intensity, short x, short y)`
- Signature: `void clear_entity_blips(void)` and `void add_entity_blip(...)`
- Purpose: Manage blip collection for entities on motion sensor
- Inputs: Entity type, intensity, screen coordinates
- Side effects: Clears or appends to `m_blips`

### `entity_blip_count()` / `entity_blip(size_t index)`
- Signature: `size_t entity_blip_count(void)` and `blip_info entity_blip(size_t index)`
- Purpose: Query blip count and retrieve specific blip data (Lua API)
- Outputs: Count (size_t) or blip struct
- Notes: Allows Lua code to iterate blips without direct vector access

### `start_draw()` / `end_draw()` / `apply_clip()`
- Signature: `void start_draw(void)`, `void end_draw(void)`, `void apply_clip(void)`
- Purpose: Bracket drawing operations; manage clipping state
- Side effects: Set/unset `m_drawing` flag; configure clipping rect

### `fill_rect()` / `frame_rect()` / `draw_text()` / `draw_image()` / `draw_shape()`
- Signature: Various (float x, y, w, h, RGBA; FontSpecifier*, Image_Blitter*, Shape_Blitter*)
- Purpose: Public Lua-facing drawing primitives with floating-point coordinates and RGBA colors
- Inputs: Geometry, colors (0.0ΓÇô1.0), font/image/shape pointers
- Side effects: Render to `m_surface` or OpenGL context

### `render_motion_sensor()` / `draw_all_entity_blips()`
- Signature: `void render_motion_sensor(short time_elapsed)` and `void draw_all_entity_blips(void)`
- Purpose: Draw motion sensor background and entity blips each frame
- Calls: Uses `draw_entity_blip()`, `draw_or_erase_unclipped_shape()`
- Notes: Protected; called from base class update loop

## Control Flow Notes
This class participates in the game frame loop via `HUD_Class::update_everything()`. Each frame:
1. Base class calls virtual `update_motion_sensor()` (updates blip state)
2. Lua code calls `start_draw()` to begin rendering
3. Lua script invokes public drawing methods (`fill_rect`, `draw_text`, etc.)
4. Lua code calls `end_draw()` to finalize
5. Protected `render_motion_sensor()` renders sensor background and blips

The Lua integration allows game themes to define HUD layouts and colors declaratively rather than in C++.

## External Dependencies
- **Includes**: `config.h` (feature flags), `HUDRenderer.h` (base class, HUD constants, world types)
- **Forward declarations**: `FontSpecifier`, `Image_Blitter`, `Shape_Blitter` (rendering primitives)
- **SDL types**: `SDL_Surface`, `SDL_Rect` (software rendering backend)
- **Inherited from base**: `HUD_Class` (update loop, motion sensor, inventory, weapon display)
- **Game engine types**: `shape_descriptor`, `screen_rectangle`, `world_distance`, `angle`, `point2d` (defined elsewhere in game engine)

---

**Notes:**
- Most protected virtual overrides (e.g., `DrawShape`, `FillRect`, `DrawText`) are **empty stubs** because Lua HUD uses the public floating-point API instead.
- Lua access to blips is read-only via `entity_blip_count()` and `entity_blip(index)`; Lua cannot directly modify blips after adding them.
