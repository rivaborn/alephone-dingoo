# Source_Files/RenderOther/screen_shared.h

## File Purpose
Shared header between screen rendering implementations (screen.cpp and screen_sdl.cpp). Defines screen dimensions, global color/view state, screen messaging, script HUD elements, and core display functions for rendering the game world, HUD, text overlays, and network status information.

## Core Responsibilities
- Define screen resolution constants for various platforms (Dingoo, standard)
- Manage global color tables (uncorrected, gamma-corrected, interface, visible)
- Maintain screen mode and field-of-view state
- Implement frame-rate display, position debugging, and message queue system
- Manage script-driven HUD elements (icons, text, colors)
- Provide on-screen text rendering (FPS, position, messages, network scores/mic status)
- Handle screen reset, zoom, and effect transitions (teleport, extravision, tunnel vision)
- Coordinate HUD and terminal rendering requests

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ScreenMessage` | struct | Queued message with expiry timer and text buffer (256 chars) |
| `ScriptHUDElement` | struct | Script-controlled HUD element: icon (RGBA 16├ù16), text (256 chars), color, SDL/OpenGL blitters |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `uncorrected_color_table`, `world_color_table`, `interface_color_table`, `visible_color_table` | `struct color_table*` | global | Color environments; pristine, gamma-corrected, interface-only (8-bit), and visible (player view) |
| `world_view` | `struct view_data*` | global | Player's camera/view state (position, angles, scales, effects) |
| `world_pixels_structure` | `struct bitmap_definition*` | global | Rendering target: dimensions and pixel row pointers |
| `screen_mode` | `struct screen_mode_data` | static | Current screen resolution, bit depth, gamma settings |
| `displaying_fps`, `frame_count`, `frame_index`, `frame_ticks[64]` | bool, short, short, int32[] | static | FPS counter state; rolling sample of 20 frame tick times |
| `ShowPosition`, `ShowScores` | bool | static | Debug/display toggles for player position and network scores |
| `HUD_RenderRequest`, `Term_RenderRequest` | bool | static | Deferred rendering flags for HUD and terminal |
| `screen_initialized` | bool | static | Initialization state |
| `bit_depth`, `interface_bit_depth` | short | static | Pixel depth (color and interface) |
| `Messages[7]`, `MostRecentMessage` | `ScreenMessage[]`, int | static | Circular message queue; most recent index for 7 concurrent messages |
| `ScriptHUDElements[MAXIMUM_NUMBER_OF_SCRIPT_HUD_ELEMENTS]` | `struct ScriptHUDElement[]` | static | Script-driven HUD element slots |

## Key Functions / Methods

### reset_screen
- **Signature:** `void reset_screen(void)`
- **Purpose:** Reset all screen state when starting a new game (map scale, view angles, effects, messages, HUD elements).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Modifies `world_view` and clears `Messages` and `ScriptHUDElements` arrays.
- **Calls:** `ResetFieldOfView()`, `reset_messages()`.
- **Notes:** Called at game start; clears teleporting/terminal effects and restores normal FOV.

### ResetFieldOfView
- **Signature:** `void ResetFieldOfView(void)`
- **Purpose:** Restore field of view based on current player extravision state.
- **Inputs:** Reads `current_player->extravision_duration`.
- **Outputs/Return:** None.
- **Side effects:** Updates `world_view->field_of_view`, `target_field_of_view`, `tunnel_vision_active`.
- **Calls:** (none directly visible)
- **Notes:** Called on respawn; deactivates tunnel vision and sets EXTRAVISION_FIELD_OF_VIEW or NORMAL_FIELD_OF_VIEW.

### zoom_overhead_map_in / zoom_overhead_map_out
- **Signature:** `bool zoom_overhead_map_in(void)`, `bool zoom_overhead_map_out(void)`
- **Purpose:** Increment or decrement overhead map zoom scale within bounds.
- **Inputs:** None.
- **Outputs/Return:** `bool` success flag (true if scale changed).
- **Side effects:** Modifies `world_view->overhead_map_scale`.
- **Calls:** (none)
- **Notes:** Clipped to `[OVERHEAD_MAP_MINIMUM_SCALE, OVERHEAD_MAP_MAXIMUM_SCALE]`.

### start_teleporting_effect / start_extravision_effect / start_tunnel_vision_effect
- **Signature:** `void start_teleporting_effect(bool out)`, `void start_extravision_effect(bool out)`, `void start_tunnel_vision_effect(bool out)` (declaration only)
- **Purpose:** Initiate screen effects for teleport, extravision, and tunnel vision.
- **Inputs:** `out` ΓÇô direction (true = activate/out, false = deactivate/in).
- **Outputs/Return:** None.
- **Side effects:** Sets render effects, field-of-view targets, or tunnel-vision flag in `world_view`.
- **Calls:** `start_render_effect()` (teleport); direct assignment (others).
- **Notes:** Extravision and tunnel-vision use gradual FOV interpolation (target-based).

### get_screen_mode
- **Signature:** `screen_mode_data* get_screen_mode(void)`
- **Purpose:** Query current screen mode configuration.
- **Inputs:** None.
- **Outputs/Return:** Pointer to `screen_mode`.
- **Side effects:** None.
- **Calls:** (none)

### game_window_is_full_screen
- **Signature:** `bool game_window_is_full_screen(void)`
- **Purpose:** Check if game window is in fullscreen mode.
- **Inputs:** Calls `alephone::Screen::instance()->hud()`.
- **Outputs/Return:** `bool` (inverse of HUD visibility).
- **Side effects:** None.
- **Calls:** `alephone::Screen::instance()->hud()` (defined elsewhere).

### change_gamma_level
- **Signature:** `void change_gamma_level(short gamma_level)`
- **Purpose:** Update gamma correction and apply to all color tables and display.
- **Inputs:** `gamma_level` (target gamma).
- **Outputs/Return:** None.
- **Side effects:** Recalculates `world_color_table`, updates visible color table, resets fade effect, resets screen mode.
- **Calls:** `gamma_correct_color_table()`, `stop_fade()`, `obj_copy()`, `assert_world_color_table()`, `change_screen_mode()`, `set_fade_effect()`.

### DisplayText
- **Signature:** `void DisplayText(short BaseX, short BaseY, const char *Text, unsigned char r=0xff, unsigned char g=0xff, unsigned char b=0xff)`
- **Purpose:** Render text at screen position with optional OpenGL path; defaults to white text with black drop shadow.
- **Inputs:** `BaseX`, `BaseY` (screen position); `Text` (string); `r`, `g`, `b` (color; default white).
- **Outputs/Return:** None.
- **Side effects:** Draws to `DisplayTextDest` (SDL surface) or OpenGL framebuffer; reads `DisplayTextFont`, `DisplayTextStyle`.
- **Calls:** `OGL_RenderText()` (if OpenGL active and conditions met), `draw_text()` (SDL fallback).
- **Notes:** Uses global `DisplayTextDest`, `DisplayTextFont`, `DisplayTextStyle` (must be set by caller). Draws shadow at offset (1,1) in black.

### update_fps_display
- **Signature:** `static void update_fps_display(SDL_Surface *s)`
- **Purpose:** Calculate and display frame rate and network latency overlay.
- **Inputs:** `s` (SDL surface to render to).
- **Outputs/Return:** None.
- **Side effects:** Updates `frame_ticks`, `frame_index`, `frame_count` rolling counters; calls `DisplayText()` to render FPS string.
- **Calls:** `SDL_GetTicks()`, `GetOnScreenFont()`, `NetGetLatency()`, `DisplayText()`.
- **Notes:** Only runs if `displaying_fps` is true and player not in terminal; resets counters when display disabled. Shows "XX.XXfps (latency ms)".

### DisplayPosition
- **Signature:** `static void DisplayPosition(SDL_Surface *s)`
- **Purpose:** Debug overlay showing player position (X, Y, Z), polygon, yaw, and pitch.
- **Inputs:** `s` (SDL surface).
- **Outputs/Return:** None.
- **Side effects:** Reads `world_view` position/angle state; renders via `DisplayText()`.
- **Calls:** `GetOnScreenFont()`, `DisplayText()`.
- **Notes:** Only renders if `ShowPosition` true. Converts world units and angles for display.

### DisplayMessages
- **Signature:** `static void DisplayMessages(SDL_Surface *s)`
- **Purpose:** Render queued screen messages and script HUD elements with text/icons.
- **Inputs:** `s` (SDL surface).
- **Outputs/Return:** None.
- **Side effects:** Decrements `Message.TimeRemaining` for each visible message; renders text and icons via `DisplayText()` and blitters.
- **Calls:** `GetOnScreenFont()`, `DisplayText()`, `OGL_IsActive()`, `_get_interface_color()`, icon blitter `Draw()`.
- **Notes:** Messages expire when `TimeRemaining <= 0`. Script HUD elements rendered above messages with optional 16├ù16 icon + text. Resets script HUD element loop index (unusual pattern).

### DisplayScores
- **Signature:** `static void DisplayScores(SDL_Surface *s)`
- **Purpose:** Render networked game leaderboard with player names, scores, latency, jitter, and error counts.
- **Inputs:** `s` (SDL surface).
- **Outputs/Return:** None.
- **Side effects:** Renders table headers and player rows via `DisplayText()`. Color-codes latency/jitter (green < 150ms, yellow < 350ms, red).
- **Calls:** `NetGetStats()`, `calculate_player_rankings()`, `calculate_ranking_text()`, `get_player_data()`, `_get_interface_color()`, `DisplayText()`.
- **Notes:** Only renders in networked games and if `ShowScores` true. DC = disconnected, centered on screen below messages. Guarded by `#if !defined(DISABLE_NETWORKING)`.

### DisplayNetMicStatus
- **Signature:** `static void DisplayNetMicStatus(SDL_Surface *s)`
- **Purpose:** Display network microphone status and speaking player indicator.
- **Inputs:** `s` (SDL surface).
- **Outputs/Return:** None.
- **Side effects:** Renders status text and icon in bottom-right corner via `DisplayText()`.
- **Calls:** `current_netgame_allows_microphone()`, `get_player_data()`, `_get_interface_color()`, `DisplayText()`.
- **Notes:** Shows "disabled" if mic off; "all"/"team" if local player speaking; speaker name if another player speaking. Only in networked games.

### screen_printf
- **Signature:** `void screen_printf(const char *format, ...)`
- **Purpose:** Queue a formatted message for screen display (7 seconds duration).
- **Inputs:** Format string and variadic arguments (printf-style).
- **Outputs/Return:** None.
- **Side effects:** Stores formatted message in circular `Messages` queue; sets `TimeRemaining` to 7 seconds.
- **Calls:** `vsnprintf()`.
- **Notes:** Circular FIFO (7-message capacity); uses safe vsnprintf to prevent buffer overflow.

### SetScriptHUDColor, SetScriptHUDText, SetScriptHUDIcon, SetScriptHUDSquare
- **Signature:** `void SetScriptHUDColor(int idx, int color)`, `void SetScriptHUDText(int idx, const char* text)`, `bool SetScriptHUDIcon(int idx, const char* text, size_t rem)`, `void SetScriptHUDSquare(int idx, int _color)`
- **Purpose:** Script API to configure HUD elements (colors, text, custom icons, or solid color squares).
- **Inputs:** `idx` (HUD slot), `color` (1ΓÇô8), `text` (string or icon descriptor), `rem` (descriptor length).
- **Outputs/Return:** SetScriptHUDIcon returns parse success.
- **Side effects:** Modifies `ScriptHUDElements[idx]`; parses icon format (hex RGBA palette + index graphic); loads SDL/OpenGL blitters.
- **Calls:** `icon::parseicon()`, `icon::seticon()`, `SDL_CreateRGBSurfaceFrom()`, `SDL_FreeSurface()`, `OGL_IsActive()`, `_get_interface_color()`.
- **Notes:** Index modulo MAXIMUM_NUMBER_OF_SCRIPT_HUD_ELEMENTS; icon parsing handles RGBA alpha channel; loads into SDL or OpenGL blitter.

### icon namespace functions (parseicon, seticon, helper functions)
- **Scope:** Static namespace `icon::{...}`
- **Purpose:** Parse and render hex-encoded icon descriptions and palette data; convert to RGBA and load into blitter.
- **Key helpers:** `nextc()`, `isadigit()`, `digit()`, `readuc()` for hex parsing.
- **Notes:** Icon format: decimal numcolors, delimiter, palette (RGB[AA] x numcolors), then 256 indices. Used by SetScriptHUDIcon.

### RequestDrawingHUD, RequestDrawingTerm
- **Signature:** `void RequestDrawingHUD(void)`, `void RequestDrawingTerm(void)`
- **Purpose:** Signal that HUD or terminal should be rendered on next frame (deferred rendering).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Set `HUD_RenderRequest` or `Term_RenderRequest` flags.
- **Calls:** (none)
- **Notes:** Decouples request from actual rendering (main display render loop consumes flags).

### GetTunnelVision, SetTunnelVision
- **Signature:** `bool GetTunnelVision(void)`, `bool SetTunnelVision(bool TunnelVisionOn)`
- **Purpose:** Query or enable/disable tunnel-vision mode.
- **Inputs:** `TunnelVisionOn` (boolean state).
- **Outputs/Return:** New tunnel-vision state.
- **Side effects:** Updates `world_view->tunnel_vision_active`; triggers effect transition.
- **Calls:** `start_tunnel_vision_effect()`.

## Control Flow Notes
This is a support/interface layer for frame-based rendering in a game engine. Screen state (color tables, view, modes) is initialized once and persisted. Per-frame rendering occurs via the `update_fps_display()`, `DisplayPosition()`, `DisplayMessages()`, etc. functions, which are called during the main rendering pass (likely from screen.cpp or screen_sdl.cpp). Messages auto-expire, HUD elements are script-configurable, and network status overlays are conditional on game mode. Fog-of-war effects (teleport, extravision, tunnel vision) update `world_view` state with smooth transitions.

## External Dependencies
- **Config & string handling:** `config.h`, `<stdarg.h>`, `snprintf.h` (platform compatibility)
- **Rendering:** `screen_drawing.h`, `Image_Blitter.h`, `OGL_Blitter.h` (blitter/image loading)
- **Networking:** `network_games.h` (game state, player rankings, net stats)
- **Console/UI:** `Console.h` (input state for overlay positioning)
- **Symbols defined elsewhere:** `uncorrected_color_table`, `world_color_table`, `interface_color_table`, `visible_color_table`, `world_view`, `world_pixels_structure`, `current_player`, `dynamic_world`, `current_player_index`, `player_in_terminal_mode()`, `View_DoFoldEffect()`, `start_render_effect()`, `OGL_MapActive`, `OGL_IsActive()`, `OGL_RenderText()`, `gamma_correct_color_table()`, `stop_fade()`, `obj_copy()`, `assert_world_color_table()`, `change_screen_mode()`, `set_fade_effect()`, `GetOnScreenFont()`, `SDL_MapRGB()`, `NetGetLatency()`, `current_netgame_allows_microphone()`, `get_player_data()`, `_get_interface_color()`, `_computer_interface_text_color`, `get_screen_mode()`, `alephone::Screen::instance()`, `local_player_index`, `game_is_networked`, `temporary` (global buffer), `MACHINE_TICKS_PER_SECOND`, `TICKS_PER_SECOND`, `dirty_terminal_view()`.
