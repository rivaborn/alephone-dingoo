# Source_Files/RenderOther/HUDRenderer_SW.h

## File Purpose
Software-rendering implementation of the HUD renderer for the Aleph One game engine. Provides a concrete subclass that renders the heads-up display (weapon panels, ammo, motion sensor, player status) using CPU-based graphics primitives instead of hardware acceleration.

## Core Responsibilities
- Implement software-rendered HUD display pipeline
- Render the motion sensor (radar) with entity tracking
- Draw weapon panels, ammo counters, and inventory display
- Provide primitive drawing operations: shapes, text, filled/framed rectangles
- Handle entity blips (radar contacts) for motion sensor
- Manage clip plane constraints for rendering regions (Windows platform)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| HUD_SW_Class | class | Software-rendering implementation of HUD display; inherits from HUD_Class |

## Global / File-Static State
None.

## Key Functions / Methods

All methods are protected virtual overrides of pure virtual methods declared in the base class `HUD_Class`. This header declares only the interface; implementations are in the corresponding `.cpp` file.

**Motion Sensor & Entity Blips:**
- `update_motion_sensor()`, `render_motion_sensor()` ΓÇö Update and render the motion sensor display
- `draw_entity_blip()`, `erase_entity_blip()` ΓÇö Draw/erase radar contacts
- `draw_or_erase_unclipped_shape()` ΓÇö Render unclipped shapes (no clip-plane constraints)

**Primitive Drawing:**
- `DrawShape()` ΓÇö Draw a shape (sprite/bitmap) with source/destination rectangles
- `DrawShapeAtXY()` ΓÇö Draw a shape at a specific screen coordinate
- `DrawText()` ΓÇö Render text with color, font, and alignment flags
- `FillRect()` ΓÇö Fill a rectangle with a solid color
- `FrameRect()` ΓÇö Draw a rectangle outline
- `DrawTexture()` ΓÇö Draw a textured element at a coordinate with size

**Clipping:**
- `SetClipPlane()` ΓÇö Define a circular clipping region (empty stub)
- `DisableClipPlane()` ΓÇö Disable clipping (empty stub)

## Control Flow Notes
This class is invoked during the HUD rendering phase of each game frame. The base class `HUD_Class::update_everything()` manages the overall update/render cycle, calling motion sensor updates, inventory/weapon panel updates, and then rendering all HUD elements via these software-rendering primitives. The SW (software) variant uses CPU-based drawing instead of GPU acceleration.

## External Dependencies
- **HUDRenderer.h** ΓÇö Base class definition; defines pure virtual methods and common HUD update logic
- **shape_descriptor**, **screen_rectangle**, **point2d** ΓÇö Types defined elsewhere (likely screen_drawing.h)
- **Platform macros** ΓÇö Windows-specific `#undef DrawText` (Windows SDK defines DrawText as a macro; this prevents collision with the virtual method)

**Notes:** All method implementations are deferred to the `.cpp` file. `SetClipPlane()` and `DisableClipPlane()` are stub implementations (likely clipping support is not critical for software rendering).
