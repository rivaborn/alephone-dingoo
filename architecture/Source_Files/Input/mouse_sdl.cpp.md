# Source_Files/Input/mouse_sdl.cpp

## File Purpose
Implements SDL-specific mouse input handling for the game engine. Manages mouse state tracking, movement normalization with configurable sensitivity/acceleration, and conversion of mouse input into game control deltas (yaw, pitch, velocity). Also handles mouse button simulation as keypresses and scroll wheel input for weapon cycling.

## Core Responsibilities
- Initialize/shutdown in-game mouse handling (`enter_mouse`, `exit_mouse`)
- Sample and normalize mouse movement deltas with per-axis sensitivity and optional acceleration
- Convert raw mouse motion into game input (yaw/pitch/velocity) using fixed-point math
- Maintain mouse state snapshot between frame samples and frame queries
- Simulate mouse buttons as SDL keycodes for input rebinding
- Track and queue scroll wheel events for weapon cycling
- Manage mouse capture/release and visibility via SDL

## Key Types / Data Structures
None (uses SDL primitives and external structures).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `mouse_active` | bool | static | Tracks whether mouse input is enabled in-game |
| `button_mask` | uint8 | static | Bitmask of enabled mouse buttons (requires release-then-press to enable) |
| `center_x`, `center_y` | int | static | Screen center coordinates for mouse recentering and delta calculation |
| `snapshot_delta_yaw` | _fixed | static | Accumulated yaw rotation from mouse movement |
| `snapshot_delta_pitch` | _fixed | static | Accumulated pitch rotation from mouse movement |
| `snapshot_delta_velocity` | _fixed | static | Accumulated forward/backward velocity from mouse movement |
| `snapshot_delta_scrollwheel` | _fixed | static | Scroll wheel accumulator for weapon cycling |

## Key Functions / Methods

### enter_mouse
- **Signature**: `void enter_mouse(short type)`
- **Purpose**: Initialize mouse input when entering in-game state
- **Inputs**: `type` ΓÇô input device type enum
- **Outputs/Return**: None
- **Side effects**: Grabs SDL mouse, disables motion events temporarily, resets all snapshots to zero, recenters mouse, sets `mouse_active = true`
- **Calls**: `SDL_WM_GrabInput()`, `SDL_EventState()`, `recenter_mouse()`
- **Notes**: Only activates if `type != _keyboard_or_game_pad`. Clears button mask to prevent accidental fire from preceding GUI clicks.

### exit_mouse
- **Signature**: `void exit_mouse(short type)`
- **Purpose**: Shutdown mouse input when leaving game
- **Inputs**: `type` ΓÇô input device type enum
- **Outputs/Return**: None
- **Side effects**: Releases SDL mouse grab, re-enables motion events, sets `mouse_active = false`
- **Calls**: `SDL_WM_GrabInput()`, `SDL_EventState()`

### recenter_mouse
- **Signature**: `void recenter_mouse(void)`
- **Purpose**: Recalculate screen center coordinates and warp mouse back to center
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Updates `center_x` and `center_y` from SDL surface dimensions, calls `SDL_WarpMouse()`
- **Calls**: `SDL_GetVideoSurface()`, `SDL_WarpMouse()`
- **Notes**: Only operates if `mouse_active == true`.

### mouse_idle
- **Signature**: `void mouse_idle(short type)`
- **Purpose**: Sample current mouse position and compute normalized movement deltas
- **Inputs**: `type` ΓÇô `_mouse_yaw_pitch` or `_mouse_yaw_velocity` (determines whether Y-axis controls pitch or velocity)
- **Outputs/Return**: None (updates static `snapshot_*` variables)
- **Side effects**: Retrieves mouse position, warps back to center, applies sensitivity scaling and optional acceleration nonlinearity, stores deltas
- **Calls**: `SDL_GetTicks()`, `SDL_GetMouseState()`, `SDL_WarpMouse()`, `TEST_FLAG()`, `PIN()` macro
- **Notes**: 
  - Uses elapsed tick count to normalize velocity (movement per millisecond in fixed-point)
  - Applies inversion if `input_preferences->modifiers & _inputmod_invert_mouse`
  - Scales X and Y independently by `sens_horizontal` and `sens_vertical`
  - If `mouse_acceleration` enabled: applies quadratic-like nonlinearity with pinning
  - If disabled: simple pinning without acceleration
  - Always clamps final deltas to `┬▒FIXED_ONE/4` (after shift)

### test_mouse
- **Signature**: `void test_mouse(short type, uint32 *flags, _fixed *delta_yaw, _fixed *delta_pitch, _fixed *delta_velocity)`
- **Purpose**: Return accumulated mouse input to game engine and apply scroll wheel weapon cycling
- **Inputs**: `type` (unused), pointers to output variables and flags word
- **Outputs/Return**: Writes `delta_yaw`, `delta_pitch`, `delta_velocity`; updates `flags` with `_cycle_weapons_*` bits
- **Side effects**: Clears all snapshots after reading; processes scroll wheel accumulator
- **Calls**: None (direct pointer writes)
- **Notes**: 
  - **Bug**: Lines 195ΓÇô197 contain incorrect pointer assignments (`delta_yaw = 0` should be `*delta_yaw = 0`); these do not update caller's variables when `mouse_active == false`.
  - Sets scroll wheel delta to zero after processing.

### mouse_buttons_become_keypresses
- **Signature**: `void mouse_buttons_become_keypresses(Uint8* ioKeyMap)`
- **Purpose**: Convert mouse button states into SDL keymap entries for input rebinding
- **Inputs**: `ioKeyMap` ΓÇô keymap array to modify
- **Outputs/Return**: None (modifies keymap in-place)
- **Side effects**: Updates 8 keymap entries (indices `SDLK_BASE_MOUSE_BUTTON` + 0ΓÇô7) with button press/release state
- **Calls**: `SDL_GetMouseState()`
- **Notes**: Button mask gates output; buttons require full release before re-enabling (prevents GUI interaction from firing weapons).

### Utility Functions
- **`hide_cursor()` / `show_cursor()`**: Simple wrappers around `SDL_ShowCursor()`.
- **`get_mouse_position(short *x, short *y)`**: Returns current absolute screen coordinates via `SDL_GetMouseState()`.
- **`mouse_still_down()`**: Returns true if left button currently pressed; calls `SDL_PumpEvents()` first.
- **`mouse_scroll(bool up)`**: Queues +1 or ΓêÆ1 to `snapshot_delta_scrollwheel` for weapon cycling.

## Control Flow Notes
Frame-based input snapshot pattern:
1. **Init**: `enter_mouse()` activates mouse capture
2. **Per-tick**: `mouse_idle()` samples and computes deltas; `mouse_buttons_become_keypresses()` updates keymap; `mouse_scroll()` queues events
3. **Per-query**: `test_mouse()` returns accumulated deltas and consumes snapshots
4. **Shutdown**: `exit_mouse()` releases capture

This decoupling allows game logic to query input at different cadence from sampling.

## External Dependencies
- **cseries.h**: `_fixed` type, `FIXED_ONE`, `FIXED_FRACTIONAL_BITS`, `PIN()` and `TEST_FLAG()` macros
- **mouse.h**: Function declarations
- **player.h**: Action flag enums (`_cycle_weapons_forward`, `_cycle_weapons_backward`, `_mouse_yaw_pitch`, `_mouse_yaw_velocity`, `_keyboard_or_game_pad`)
- **shell.h**: Input device type enums
- **preferences.h**: `input_preferences` global struct (sensitivity, modifiers, acceleration settings)
- **SDL.h**: `SDL_WM_GrabInput()`, `SDL_EventState()`, `SDL_GetVideoSurface()`, `SDL_GetTicks()`, `SDL_GetMouseState()`, `SDL_WarpMouse()`, `SDL_ShowCursor()`
