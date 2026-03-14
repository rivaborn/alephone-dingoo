# Source_Files/Input/ISp_Support.cpp
## File Purpose
Implements InputSprocket (ISp) device support for the Marathon game engine on macOS. Maps hardware input (keyboards, mice, joysticks) to game actions through virtual input elements. Handles both global networked actions and local machine-only events (quit, pause, settings).

## Core Responsibilities
- Initialize/shutdown InputSprocket framework and virtual input elements
- Define and register 44 input actions: 21 global (networked), 23 local, and 4 analog axes
- Poll input devices each frame and convert device state to action flags
- Enforce refractory periods on local-event buttons to prevent rapid repeat triggering
- Support configuration UI and device activation/deactivation
- Handle Carbon API compatibility (disable ISp under Carbon)

## External Dependencies
- **Framework includes (macOS):** `InputSprocket.h`, `CursorDevices.h`, `Traps.h` (weak-linked; disabled on Carbon)
- **Project includes:** `world.h`, `map.h`, `player.h`, `preferences.h`, `LocalEvents.h`, `ISp_Support.h`, `macintosh_cseries.h`, `math.h`
- **Defined elsewhere:** `input_preferences` (global struct), `mask_in_absolute_positioning_information()`, `PostLocalEvent()`, `temporary` (formatting buffer for csprintf), action flag enums (_left_trigger_state, _moving_forward, etc.)

# Source_Files/Input/ISp_Support.h
## File Purpose
Header file declaring the public interface for InputSprocket (ISp) support in the Marathon/Aleph One game engine. Provides lifecycle management and control configuration for an older macOS input abstraction API.

## Core Responsibilities
- Initialize and shut down the InputSprocket subsystem
- Start and stop input event monitoring during gameplay
- Query input device state (keyboard vs. other controllers)
- Test and enumerate available InputSprocket input elements
- Configure Marathon-specific control bindings for discovered input devices

## External Dependencies
- InputSprocket API (macOS-specific; implementation likely in a paired .c/.cpp file)
- Standard C library (`bool`, `void`, `long` types)

# Source_Files/Input/joystick.h
## File Purpose
Header file for joystick input handling in the Aleph One game engine. Provides interface to initialize joystick devices, translate button presses into keyboard events, and process analog axis inputs for use in the input system.

## Core Responsibilities
- Joystick device initialization and cleanup
- Translate joystick button presses into keyboard events for the keymap system
- Process analog joystick axis data (sticks, triggers)
- Define constants for mapping joystick buttons to key indices in the global keymap array

## External Dependencies
- **SDL (Simple DirectMedia Layer)** ΓÇô SDLK_* constants and `Uint8` type indicate SDL integration for cross-platform joystick support
- Global keymap array ΓÇô defined elsewhere; modified by `joystick_buttons_become_keypresses()`
- Aleph One game engine runtime ΓÇô initialization/shutdown hooks

# Source_Files/Input/joystick_sdl.cpp
## File Purpose
Bridges SDL joystick hardware input to Aleph One's action flag system. Manages joystick initialization, button-to-keypress mapping, and analog axis conversion with configurable sensitivity and pulse-modulation for strafing.

## Core Responsibilities
- Initialize/cleanup SDL joystick subsystem via `enter_joystick()` and `exit_joystick()`
- Poll joystick buttons and map them to virtual keycode range (`SDLK_BASE_JOYSTICK_BUTTON` + button index)
- Poll joystick axes (strafe, velocity, yaw, pitch) and scale via user preferences
- Apply dead zones (axis bounds clipping) and sensitivity multipliers
- Implement pulse-modulation for strafing (3 intensity bands control duty cycle, avoiding smooth analog strafing)
- Maintain joystick active state flag

## External Dependencies
- **SDL library**: `SDL.h` (SDL_Joystick, SDL_JoystickEventState, SDL_JoystickOpen/Close, SDL_JoystickGetButton/Axis, SDL_JoystickNumButtons)
- **player.h**: `mask_in_absolute_positioning_information()` (defined elsewhere; encodes yaw/pitch/velocity into action flags)
- **preferences.h**: global `input_preferences` pointer (holds axis mappings, sensitivities, bounds, joystick ID)
- **joystick.h**: header declaring this module's public interface
- **\<algorithm\>**: `std::max()` for button iteration

# Source_Files/Input/mouse.cpp
## File Purpose
Provides mouse input handling for the Marathon game engine on Mac (Carbon) platforms. Captures mouse movement and button events, converts them to game action flags and view delta values (yaw, pitch, velocity), and applies user-preference settings like sensitivity and acceleration.

## Core Responsibilities
- Initialize and teardown mouse input event handling via Carbon Event Manager
- Capture raw mouse position and button events from Carbon event stream
- Calculate frame-by-frame mouse deltas and convert to game action values (yaw, pitch, velocity)
- Apply input preferences (mouse sensitivity, Y-axis inversion, acceleration curves) to raw deltas
- Map mouse button presses to game triggers (left/right weapon fire) and scroll wheel to weapon cycling
- Handle mouse warping (centering) to enable relative-motion tracking
- Provide thread-safe communication between event handler (main thread) and input sampling (separate thread via critical regions)

## External Dependencies
- **Marathon engine includes:**
  - world.h ΓÇö integer_to_fixed conversion, world coordinate types.
  - map.h ΓÇö player-related structures, game state queries.
  - player.h ΓÇö get_absolute_pitch_range() for pitch clamping.
  - shell.h ΓÇö interface declarations, get_game_state(), process_screen_click().
  - interface.h ΓÇö FrontWindow(), screen_window (Carbon).
  - preferences.h ΓÇö input_preferences global (sensitivity, acceleration, button mappings, modifier flags).
  - Logging.h ΓÇö dprintf() macro (conditional debug logging).

- **Mac/Carbon APIs:**
  - ApplicationServices.h (TARGET_API_MAC_CARBON) ΓÇö Carbon Event Manager, Point type, critical regions, dynamic function loading.
  - CoreGraphics (CGDirectDisplay.h) ΓÇö CGGetLastMouseDelta, CGWarpMouseCursorPosition (function pointers).
  - macintosh_cseries.h ΓÇö fixed-point types (_fixed), utility macros (PIN, TEST_FLAG).

- **External symbols defined elsewhere:**
  - `input_preferences`, `dynamic_world` ΓÇö globals from preferences.cpp and map.cpp.
  - `process_screen_click()`, `get_game_state()`, `FrontWindow()`, `GetGlobalMouse()`, `Button()` ΓÇö declared in shell.h or Carbon headers.

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

## External Dependencies
- **Custom types:** `_fixed` (fixed-point), `uint32`, used for deterministic input calculations
- **SDL library:** Conditionally compiled; `Uint8`, `SDLK_BASE_MOUSE_BUTTON` from SDL headers
- **Input type enum:** `short type` parameter references an input device type defined elsewhere
- **Action flags enum/constants:** Referenced by test_mouse output but not defined here

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

## External Dependencies
- **cseries.h**: `_fixed` type, `FIXED_ONE`, `FIXED_FRACTIONAL_BITS`, `PIN()` and `TEST_FLAG()` macros
- **mouse.h**: Function declarations
- **player.h**: Action flag enums (`_cycle_weapons_forward`, `_cycle_weapons_backward`, `_mouse_yaw_pitch`, `_mouse_yaw_velocity`, `_keyboard_or_game_pad`)
- **shell.h**: Input device type enums
- **preferences.h**: `input_preferences` global struct (sensitivity, modifiers, acceleration settings)
- **SDL.h**: `SDL_WM_GrabInput()`, `SDL_EventState()`, `SDL_GetVideoSurface()`, `SDL_GetTicks()`, `SDL_GetMouseState()`, `SDL_WarpMouse()`, `SDL_ShowCursor()`


