# Source_Files/RenderMain/Crosshairs_SDL.cpp

## File Purpose
Implements SDL-based rendering of crosshairs centered on the player's viewport. Supports two crosshair shapes (traditional cross and circle approximated as octagon) with configurable color, thickness, and distance from center. Part of the Aleph One engine's heads-up display system.

## Core Responsibilities
- Manage crosshair active/inactive state via getter and setter functions
- Render crosshairs to an SDL surface at screen center
- Support RealCrosshairs shape (four-bar cross using filled rectangles)
- Support Circle shape (twelve-segment octagon approximation using line drawing)
- Map crosshair color from RGB to SDL pixel format
- Calculate crosshair dimensions based on configuration (thickness, length, distance from center)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CrosshairData` | struct (external) | Configuration data: color, thickness, distance from center, length, shape type, opacity |
| `world_point2d` | struct (external) | 2D point for line endpoints; has `x` and `y` fields |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_Crosshairs_IsActive` | bool | static (file scope) | Toggles whether crosshairs are rendered on next frame |

## Key Functions / Methods

### Crosshairs_IsActive
- Signature: `bool Crosshairs_IsActive(void)`
- Purpose: Query current crosshair activation state
- Inputs: None
- Outputs/Return: `true` if active, `false` if inactive
- Side effects: None
- Calls: None

### Crosshairs_SetActive
- Signature: `bool Crosshairs_SetActive(bool NewState)`
- Purpose: Enable or disable crosshair rendering
- Inputs: `NewState` ΓÇô desired active state
- Outputs/Return: The new state value (assignment return)
- Side effects: Modifies file-static `_Crosshairs_IsActive`
- Calls: None

### Crosshairs_Render
- Signature: `bool Crosshairs_Render(SDL_Surface *s)`
- Purpose: Draw crosshairs centered on the supplied SDL surface
- Inputs: `s` ΓÇô SDL surface to draw into (expected to be the main viewport)
- Outputs/Return: `true` if rendered, `false` if inactive
- Side effects: Writes pixels to SDL surface via `SDL_FillRect()` (RealCrosshairs) or `draw_line()` (Circle)
- Calls: `GetCrosshairData()`, `SDL_MapRGB()`, `SDL_FillRect()`, `draw_line()`
- Notes:
  - Early return if `_Crosshairs_IsActive` is false
  - Calculates screen center as `(s->w / 2 - 1, s->h / 2 - 1)`
  - RealCrosshairs path: four `SDL_FillRect()` calls for left, right, top, bottom bars
  - Circle path: pre-computes 8 octagon point coordinates; unrolled nested loop draws 12 line segments (3 segments ├ù 2 axes ├ù 2 quadrants), each segment composed of one vertical + one diagonal + one horizontal line segment

## Control Flow Notes
Pure render-phase function with no state updates or initialization. Called once per frame from the main rendering loop if crosshairs are enabled. No dependencies on update or physics; purely visual.

## External Dependencies
- **SDL**: `SDL_Surface`, `SDL_Rect`, `SDL_FillRect()`, `SDL_MapRGB()`
- **Crosshairs.h**: `CrosshairData` struct, `GetCrosshairData()` function
- **screen_drawing.h**: `draw_line()` function for octagon segment drawing
- **world.h**: `world_point2d` struct for line endpoint coordinates
- **cseries.h**: Standard type definitions, SDL includes, macros
