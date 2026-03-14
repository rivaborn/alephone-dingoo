# Source_Files/RenderMain/Rasterizer_SW.h

## File Purpose
Software rasterizer implementation that provides texture rendering for a game engine. Inherits from `RasterizerClass` base to plug into a polymorphic rendering pipeline. Implementations delegate to legacy `scottish_textures` module.

## Core Responsibilities
- Provide software-based rasterizer as a `RasterizerClass` subclass
- Manage view data and screen bitmap pointers for rendering
- Implement polygon texture rasterization (horizontal and vertical)
- Implement rectangle texture rasterization
- Act as an abstraction layer enabling swappable rasterizer backends (software vs. OpenGL)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Rasterizer_SW_Class | class | Software rasterizer implementation; inherits from RasterizerClass |

## Global / File-Static State
None.

## Key Functions / Methods

### SetView
- Signature: `void SetView(view_data& View)`
- Purpose: Configure the rasterizer with view data before rendering begins
- Inputs: Reference to view_data struct containing camera/viewport info
- Outputs/Return: None (void)
- Side effects: Updates member pointer `view`
- Calls: None
- Notes: Must be called before any rendering calls; documented as mandatory precondition

### texture_horizontal_polygon
- Signature: `void texture_horizontal_polygon(polygon_definition& textured_polygon)`
- Purpose: Rasterize a horizontally-oriented textured polygon
- Inputs: Reference to polygon_definition with geometry and texture data
- Outputs/Return: None (void)
- Side effects: Writes to screen bitmap (via `screen` member)
- Calls: Implementation in `scottish_textures.c`
- Notes: Virtual override of base class method

### texture_vertical_polygon
- Signature: `void texture_vertical_polygon(polygon_definition& textured_polygon)`
- Purpose: Rasterize a vertically-oriented textured polygon
- Inputs: Reference to polygon_definition
- Outputs/Return: None (void)
- Side effects: Writes to screen bitmap
- Calls: Implementation in `scottish_textures.c`
- Notes: Virtual override of base class method

### texture_rectangle
- Signature: `void texture_rectangle(rectangle_definition& textured_rectangle)`
- Purpose: Rasterize a textured rectangle (axis-aligned quadrilateral)
- Inputs: Reference to rectangle_definition
- Outputs/Return: None (void)
- Side effects: Writes to screen bitmap
- Calls: Implementation in `scottish_textures.c`
- Notes: Virtual override of base class method

## Control Flow Notes
This class integrates into a rendering pipeline where `RasterizerClass` defines a polymorphic interface for swappable backends. Typical frame flow: (1) instantiate rasterizer, (2) call `SetView()` with current camera data, (3) call `texture_*()` methods for each visible polygon/rectangle, (4) implementation in `scottish_textures.c` performs actual scanline fill. Base class also defines `SetForeground()` / `SetForegroundView()` / `Begin()` / `End()` hooks (not overridden here), suggesting those may be handled by scottish_textures or parent logic.

## External Dependencies
- **Rasterizer.h**: Base class `RasterizerClass`
- **render.h**: (via Rasterizer.h) Likely defines view_data, polygon_definition, rectangle_definition, bitmap_definition
- **scottish_textures.c**: Contains implementation of all three texture_* methods
- External types used but not defined here: `view_data`, `bitmap_definition`, `polygon_definition`, `rectangle_definition`
