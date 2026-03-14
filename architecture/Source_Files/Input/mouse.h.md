# Source_Files/Input/mouse.h

## File Purpose
Declares the mouse input subsystem interface for Aleph One (a game engine). Provides functions to initialize, poll, and manage mouse input, with SDL-specific helpers to simulate keyboard events from mouse buttons and scroll actions.

## Core Responsibilities
- Lifecycle management: initialize (enter_mouse) and shutdown (exit_mouse) mouse input
- Per-frame mouse state polling and input processing (test_mouse)
- Idle state handling for mouse input (mouse_idle)
- Mouse position recentering to screen center (recenter_mouse)
- SDL-specific: map mouse buttons to keyboard events (mouse_buttons_become_keypresses)
- SDL-specific: handle mouse scroll wheel input (mouse_scroll)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `uint32` | typedef | Bitfield for action flags output from test_mouse |
| `_fixed` | typedef | Fixed-point number type for rotation/velocity deltas |
| `Uint8` | SDL typedef | Byte array for key map state in SDL helper |

## Global / File-Static State
None.

## Key Functions / Methods

### enter_mouse
- **Signature:** `void enter_mouse(short type)`
- **Purpose:** Initialize mouse input subsystem for a given input type/device
- **Inputs:** `type` ΓÇô input device type (enum/constant, defined elsewhere)
- **Outputs/Return:** None
- **Side effects:** Acquires mouse device, may set up SDL event handlers if SDL is enabled
- **Calls:** None visible (implementation in .c file)
- **Notes:** Paired with exit_mouse for cleanup

### test_mouse
- **Signature:** `void test_mouse(short type, uint32 *action_flags, _fixed *delta_yaw, _fixed *delta_pitch, _fixed *delta_velocity)`
- **Purpose:** Poll mouse state and compute input deltas for the current frame
- **Inputs:** `type` ΓÇô input device type
- **Outputs/Return:** 
  - `action_flags` ΓÇô pointer to bitfield updated with button/action states
  - `delta_yaw`, `delta_pitch` ΓÇô rotation deltas (fixed-point)
  - `delta_velocity` ΓÇô movement velocity delta (fixed-point)
- **Side effects:** Reads mouse position/button state; may reset cursor position
- **Calls:** None visible
- **Notes:** Called once per game frame; uses fixed-point for deterministic network sync

### exit_mouse
- **Signature:** `void exit_mouse(short type)`
- **Purpose:** Shutdown mouse input and release device resources
- **Inputs:** `type` ΓÇô input device type
- **Outputs/Return:** None
- **Side effects:** Releases mouse device, disables SDL event handlers if enabled
- **Calls:** None visible
- **Notes:** Inverse of enter_mouse

### mouse_idle
- **Signature:** `void mouse_idle(short type)`
- **Purpose:** Handle idle/pause state for mouse input (e.g., when game is not active)
- **Inputs:** `type` ΓÇô input device type
- **Outputs/Return:** None
- **Side effects:** May pause input polling or reset mouse state
- **Calls:** None visible

### recenter_mouse
- **Signature:** `void recenter_mouse(void)`
- **Purpose:** Reset mouse cursor to screen center to prevent drifting out of window
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Moves cursor; may trigger system mouse events
- **Calls:** None visible
- **Notes:** Called per-frame in first-person view to enable unbounded rotation

### mouse_buttons_become_keypresses (SDL only)
- **Signature:** `void mouse_buttons_become_keypresses(Uint8* ioKeyMap)`
- **Purpose:** Map SDL mouse button states into keyboard key map (SDL hack for input abstraction)
- **Inputs:** `ioKeyMap` ΓÇô pointer to SDL key state array (in/out)
- **Outputs/Return:** None (modifies key map in place)
- **Side effects:** Updates key map array with simulated key presses from mouse buttons
- **Calls:** None visible
- **Notes:** Conditional SDL-only feature; maps button 1 to pseudo-keysym 65 (SDLK_BASE_MOUSE_BUTTON)

### mouse_scroll (SDL only)
- **Signature:** `void mouse_scroll(bool up)`
- **Purpose:** Handle mouse scroll wheel events as input actions
- **Inputs:** `up` ΓÇô true if scrolling up, false if down
- **Outputs/Return:** None
- **Side effects:** May trigger weapon cycling or other scroll-mapped actions
- **Calls:** None visible
- **Notes:** SDL-specific; direction-based (up/down)

## Control Flow Notes
Typical lifecycle is `enter_mouse()` ΓåÆ repeated `test_mouse()` per frame ΓåÆ `mouse_idle()` during pause ΓåÆ `exit_mouse()` on shutdown. Mouse position recentering (recenter_mouse) is called per-frame during active play to enable unconstrained first-person rotation. SDL-specific helpers (mouse_buttons_become_keypresses, mouse_scroll) integrate into the broader SDL event loop (defined elsewhere).

## External Dependencies
- **Custom types:** `_fixed` (fixed-point), `uint32`, used for deterministic input calculations
- **SDL library:** Conditionally compiled; `Uint8`, `SDLK_BASE_MOUSE_BUTTON` from SDL headers
- **Input type enum:** `short type` parameter references an input device type defined elsewhere
- **Action flags enum/constants:** Referenced by test_mouse output but not defined here
