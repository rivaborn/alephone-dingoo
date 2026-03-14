# Source_Files/RenderMain/Crosshairs.cpp

## File Purpose
Implements crosshair rendering for the game HUD, supporting multiple shapes (traditional cross or circular octagon) with configurable size, thickness, and color. Uses Quickdraw graphics API to draw in 2D screen space, centered on the viewport.

## Core Responsibilities
- Manage global crosshairs active/inactive state via static variable
- Render crosshairs at viewport center in two shapes: RealCrosshairs (perpendicular lines) or Circle (octagon approximation)
- Preserve and restore graphics context state (pen, colors) during rendering
- Provide three rendering overloads (context + rect, context only, rect only)
- Handle pen positioning and line drawing for different crosshair geometries

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CrosshairData | struct (from header) | Configuration: color, thickness, distance from center, length, shape, opacity |
| Rect | struct (from Quickdraw) | Viewport rectangle (top, left, bottom, right) |
| RGBColor | struct (from Quickdraw) | RGB color value |
| PenState | struct (from Quickdraw) | Pen configuration snapshot |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_Crosshairs_IsActive` | bool | static (file) | Toggle for whether crosshairs are rendered |

## Key Functions / Methods

### Crosshairs_IsActive
- Signature: `bool Crosshairs_IsActive()`
- Purpose: Query current active state
- Inputs: None
- Outputs/Return: `_Crosshairs_IsActive`
- Side effects: None
- Calls: None
- Notes: Simple getter

### Crosshairs_SetActive
- Signature: `bool Crosshairs_SetActive(bool NewState)`
- Purpose: Toggle crosshairs on/off
- Inputs: `NewState` (desired state)
- Outputs/Return: Updated `_Crosshairs_IsActive`
- Side effects: Modifies static state
- Calls: None
- Notes: Coerces input to bool (handles non-zero/zero)

### Crosshairs_Render (primary)
- Signature: `bool Crosshairs_Render(Rect &ViewRect)`
- Purpose: Render crosshairs at viewport center in current graphics context
- Inputs: `ViewRect` ΓÇô screen rectangle defining viewport bounds
- Outputs/Return: `_Crosshairs_IsActive`
- Side effects: Modifies graphics context (drawing, pen state); allocates/reads CrosshairData
- Calls: `GetCrosshairData()`, `GetPenState()`, `GetBackColor()`, `GetForeColor()`, `SetPenState()`, `RGBBackColor()`, `RGBForeColor()`, `PenNormal()`, `PenSize()`, `MoveTo()`, `Line()`, `LineTo()`
- Notes: 
  - Returns early if not active
  - Saves/restores all pen and color state
  - Center offset by (-1, -1) to handle even pen widths correctly
  - Handles two shapes: `CHShape_RealCrosshairs` (4 perpendicular lines) and `CHShape_Circle` (12-segment octagon using nested loops)
  - Octagon computed in `OctaPoints[2][6]` with 6 x,y pairs per axis

### Crosshairs_Render (with context and rect)
- Signature: `bool Crosshairs_Render(GrafPtr Context, Rect &ViewRect)`
- Purpose: Render crosshairs in a specified graphics port
- Inputs: `Context` (target graphics port), `ViewRect` (viewport rect)
- Outputs/Return: Result from primary `Crosshairs_Render()`
- Side effects: Pushes/pops graphics context
- Calls: `GetPort()`, `SetPort()`, `Crosshairs_Render(Rect &ViewRect)`
- Notes: Wrapper managing context stack

### Crosshairs_Render (with context only)
- Signature: `bool Crosshairs_Render(GrafPtr Context)`
- Purpose: Render crosshairs using the full bounds of a graphics port
- Inputs: `Context` (target graphics port)
- Outputs/Return: Result from two-argument overload
- Side effects: Queries port bounds via `GetPortBounds()`
- Calls: `GetPortBounds()`, `Crosshairs_Render(GrafPtr, Rect &)`
- Notes: Convenience overload; extracts rect from port

## Control Flow Notes
**Render phase**: Called during frame rendering (likely from a HUD/UI layer) to draw the center-screen aiming reticle. State is pushed before drawing and restored afterward, so no persistent graphics context corruption. Early exit if `_Crosshairs_IsActive == false` avoids unnecessary work. The octagon rendering (CHShape_Circle) uses nested loops to draw 12 line segments with varying pen sizes, approximating a circular shape using only horizontal, vertical, and diagonal line primitives (suitable for software rasterization).

## External Dependencies
- **Quickdraw API** (Mac/legacy): `GetPenState`, `SetPenState`, `RGBForeColor`, `RGBBackColor`, `PenNormal`, `PenSize`, `MoveTo`, `LineTo`, `Line`, `GetPort`, `SetPort`, `GetPortBounds`
- **cseries.h**: Core type definitions and macros
- **Crosshairs.h**: `CrosshairData` struct, shape enums (`CHShape_RealCrosshairs`, `CHShape_Circle`)
- **Preferences system** (defined elsewhere): `GetCrosshairData()` ΓÇô retrieves crosshair configuration
