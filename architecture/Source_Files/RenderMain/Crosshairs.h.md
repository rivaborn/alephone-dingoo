# Source_Files/RenderMain/Crosshairs.h

## File Purpose
Header file defining the interface for crosshair rendering in the Aleph One game engine. Provides configuration structures and functions to manage crosshair appearance, visibility state, and rendering across multiple graphics backends (classic Macintosh QuickDraw and SDL).

## Core Responsibilities
- Define `CrosshairData` structure holding visual properties (color, thickness, opacity, shape)
- Expose functions to query and toggle crosshair active state
- Provide configuration dialog for user customization
- Declare rendering entry points for multiple graphics contexts (Mac QD and SDL)
- Manage crosshair data lifecycle (retrieval from preferences)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CrosshairData` | struct | Holds all crosshair visual configuration: color, thickness, offset from center, length, shape type (real/circle), opacity, and pre-calculated GL colors |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Crosshair active flag | bool (inferred) | Global/singleton | Tracks whether crosshairs should be rendered |
| Crosshair data | `CrosshairData` | Singleton (from preferences) | Current crosshair configuration loaded from persistent preferences |

## Key Functions / Methods

### Configure_Crosshairs
- **Signature:** `bool Configure_Crosshairs(CrosshairData &Data)`
- **Purpose:** Launch interactive UI dialog for the player to customize crosshair appearance
- **Inputs:** Reference to `CrosshairData` struct to populate
- **Outputs/Return:** `true` if user confirmed, `false` if canceled; modifies passed struct only on success
- **Side effects:** May display platform-specific UI dialog
- **Calls:** Implemented in `PlayerDialogs.c`
- **Notes:** Non-destructive on cancel; caller should only apply changes on `true` return

### GetCrosshairData
- **Signature:** `CrosshairData& GetCrosshairData()`
- **Purpose:** Retrieve the current active crosshair configuration from persistent preferences
- **Outputs/Return:** Reference to the in-memory `CrosshairData` struct
- **Side effects:** None visible (reads from preferences cache)
- **Calls:** Implemented in `preferences.c`
- **Notes:** Returns a reference; modifications affect the global state

### Crosshairs_IsActive
- **Signature:** `bool Crosshairs_IsActive()`
- **Purpose:** Query whether crosshairs are currently enabled for rendering
- **Outputs/Return:** `true` if active, `false` if inactive

### Crosshairs_SetActive
- **Signature:** `bool Crosshairs_SetActive(bool NewState)`
- **Purpose:** Toggle crosshair visibility on/off
- **Inputs:** `NewState` ΓÇö desired active state
- **Outputs/Return:** `true` if successful (return value semantics not explicit)

### Crosshairs_Render
- **Signature:** 
  - `bool Crosshairs_Render(Rect &ViewRect)` (Mac)
  - `bool Crosshairs_Render(GrafPtr Context)` (Mac)
  - `bool Crosshairs_Render(GrafPtr Context, Rect &ViewRect)` (Mac)
  - `bool Crosshairs_Render(SDL_Surface *s)` (SDL)
- **Purpose:** Draw crosshairs to the specified graphics context, centered in the provided view rectangle
- **Inputs:** Graphics context and optional view rectangle (falls back to context bounds if not provided)
- **Outputs/Return:** `bool` (success/failure not explicitly documented)
- **Side effects:** Modifies framebuffer via graphics context; uses current crosshair data from preferences
- **Notes:** Compilation varies by platform (Mac QD vs SDL); multiple overloads provide flexibility for caller context

## Control Flow Notes
Crosshairs participate in the render phase of each frame loop. Configuration can be triggered asynchronously via UI dialogs. The module queries cached crosshair data from preferences (loaded at startup) and respects the active state flag to conditionally render. Rendering calls are invoked near the end of the frame after game world and HUD elements.

## External Dependencies
- **Types:** `RGBColor` (color primitive, defined elsewhere), `Rect` (rectangle, likely in Mac toolbox or SDL), `GrafPtr` (Mac QuickDraw graphics port), `SDL_Surface` (SDL graphics surface)
- **Modules:** `PlayerDialogs.c` (UI dialog implementation), `preferences.c` (preference storage/retrieval)
- **Platform abstractions:** Conditional compilation (`#if defined(mac)` / `#elif defined(SDL)`) for graphics backend selection
