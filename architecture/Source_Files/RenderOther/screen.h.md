# Source_Files/RenderOther/screen.h

## File Purpose
Header for the game engine's screen and display management system. Defines a singleton `Screen` class for managing resolution, rendering surfaces, HUD layout, and color tables. Declares functions for rendering, effects (gamma, tunnel vision, teleporting), and overlay HUD elements.

## Core Responsibilities
- Screen mode enumeration and selection (resolution management)
- Geometry calculation for 3D view, HUD, map, and terminal rendering areas
- Color table (palette) management and CLUT (Color Look-Up Table) animation
- Visual effects (teleporting, extravision, tunnel vision, gamma correction)
- Screen dumping (screenshots) and buffer management
- Overhead map display control and zoom
- Script-controlled HUD element rendering (text, icons, colored squares)
- OpenGL acceleration mode support

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Screen` | class (C++ singleton in `alephone` namespace) | Encapsulates screen state, available modes, and dimension queries; provides static accessor. |
| Screen sizes | enum | Symbolic constants for viewport scales: `_50_percent`, `_75_percent`, `_100_percent`, `_full_screen`. |
| Hardware acceleration codes | enum | `_no_acceleration`, `_opengl_acceleration`. |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `world_color_table` | `struct color_table*` | global (extern) | Palette for world/game view rendering. |
| `visible_color_table` | `struct color_table*` | global (extern) | Palette for visible surfaces. |
| `interface_color_table` | `struct color_table*` | global (extern) | Palette for HUD/UI rendering. |
| `Screen::m_instance` | `Screen` | static singleton | Global screen instance. |
| `Screen::m_modes` | `std::vector<std::pair<int,int>>` | instance | Available display resolutions (width, height). |
| `Screen::m_initialized` | `bool` | instance | Initialization flag. |

## Key Functions / Methods

### Screen::Initialize
- Signature: `void Initialize(screen_mode_data* mode)`
- Purpose: Set up screen with the given mode configuration.
- Inputs: `mode` pointer to screen mode data.
- Outputs/Return: None.
- Side effects: Initializes `m_initialized`, populates `m_modes`.
- Calls: Not inferable.
- Notes: Singleton initialization; assumes `m_instance` already exists.

### Screen::GetModes
- Signature: `const std::vector<std::pair<int,int>>& GetModes()`
- Purpose: Return the list of supported screen resolutions.
- Inputs: None.
- Outputs/Return: Const reference to vector of (width, height) pairs.
- Side effects: None.
- Calls: None.

### Screen::FindMode
- Signature: `int FindMode(int width, int height)`
- Purpose: Look up a screen mode index by dimensions.
- Inputs: `width`, `height`.
- Outputs/Return: Mode index if found, `-1` if not found.
- Side effects: None.
- Calls: None.
- Notes: Linear search; `-1` indicates mode not available.

### Screen::height / width / window_height / window_width
- Signature: `int height()`, `int width()`, `int window_height()`, `int window_width()`
- Purpose: Query current render dimensions.
- Inputs: None.
- Outputs/Return: Dimension in pixels.
- Side effects: None.
- Calls: Not inferable.
- Notes: `window_*` likely includes OS window chrome; `height/width` are viewport dimensions.

### Screen::*_rect methods
- Signature: `SDL_Rect window_rect()`, `view_rect()`, `map_rect()`, `term_rect()`, `hud_rect()`
- Purpose: Return bounding rectangles for different UI regions.
- Inputs: None.
- Outputs/Return: `SDL_Rect` with position and size.
- Side effects: None.
- Calls: Not inferable.

### render_screen
- Signature: `void render_screen(short ticks_elapsed)`
- Purpose: Main per-frame rendering call; draws world, HUD, map, effects.
- Inputs: `ticks_elapsed` time delta in game ticks.
- Outputs/Return: None.
- Side effects: Writes to screen/frame buffer; applies visual effects.
- Calls: Not inferable.
- Notes: Entry point for frame rendering; effects (gamma, tunnel vision, etc.) applied during or after.

### change_screen_mode
- Signature: `void change_screen_mode(struct screen_mode_data *mode, bool redraw)`
- Purpose: Switch to a new resolution and optionally redraw.
- Inputs: `mode` new screen configuration; `redraw` whether to force redraw.
- Outputs/Return: None.
- Side effects: Changes display resolution, modifies frame buffer geometry.
- Calls: Not inferable.

### reset_screen
- Signature: `void reset_screen()`
- Purpose: Reset screen state when starting a new game (added in Mar 2000).
- Inputs: None.
- Outputs/Return: None.
- Side effects: Clears screen-related game state.
- Calls: Not inferable.

### Teleport/Extravision/Tunnel-Vision Effects
- Signature: `void start_teleporting_effect(bool out)`, `void start_extravision_effect(bool out)`, `bool GetTunnelVision()`, `bool SetTunnelVision(bool TunnelVisionOn)`
- Purpose: Apply or query special visual effects.
- Inputs: `out` flag indicates enter/exit effect; `TunnelVisionOn` for tunnel vision toggle.
- Outputs/Return: `bool` for tunnel vision getter.
- Side effects: Modifies rendering state/filters applied during `render_screen()`.
- Calls: Not inferable.

### Script HUD Element Functions
- Signatures: `void SetScriptHUDColor(int idx, int color)`, `void SetScriptHUDText(int idx, const char* text)`, `bool SetScriptHUDIcon(int idx, const char* icon, size_t length)`, `void SetScriptHUDSquare(int idx, int color)`
- Purpose: Manage up to 6 overlay HUD elements (text, icons, colored squares) for scripting/modding.
- Inputs: `idx` [0,5], color index, text pointer, icon data.
- Outputs/Return: `bool` for icon set (success/failure).
- Side effects: Updates internal HUD state; rendered in next frame.
- Calls: Not inferable.
- Notes: Text or icon `NULL`/`""` removes element; maximum 6 elements; spacing defined by `SCRIPT_HUD_ELEMENT_SPACING`.

### Color/Gamma Functions
- Signature: `void initialize_gamma()`, `void restore_gamma()`, `void change_screen_clut(struct color_table*)`, `void animate_screen_clut(struct color_table*, bool full_screen)`, `void build_direct_color_table(struct color_table*, short bit_depth)`, `void change_gamma_level(short gamma_level)`
- Purpose: Manage palette, color rendering, and gamma correction.
- Inputs: Color table pointer, bit depth, gamma level.
- Outputs/Return: None.
- Side effects: Modifies display color lookup or gamma state.
- Calls: Not inferable.

## Control Flow Notes
- **Initialization**: `Initialize()` or `change_screen_mode()` at engine startup to set resolution and populate available modes.
- **Per-frame**: `render_screen(ticks_elapsed)` called each frame to render world, HUD, map, and apply effects.
- **Effects**: Teleport, extravision, tunnel vision, and gamma effects are toggled via their respective functions; applied during or after `render_screen()`.
- **Game start**: `reset_screen()` called when starting a new game to clear screen state.
- **Resolution change**: `change_screen_mode()` or `toggle_fullscreen()` update geometry and trigger redraw.

## External Dependencies
- Standard library: `<utility>`, `<vector>`
- SDL types: `SDL_Rect` (for rectangles)
- Forward-declared: `screen_mode_data` struct (defined elsewhere)
- Extern globals: `color_table` structures for world, visible, and interface rendering
- Notes: Code predates C++11 (uses `std::pair<int, int>` with old space syntax); copyright 1991ΓÇô2001, Bungie/Aleph One project.
