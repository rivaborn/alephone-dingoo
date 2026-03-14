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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SDL_Joystick | struct (SDL) | Opaque joystick device handle |
| action flags | int (bitfield from player.h) | Encoded movement/view commands (_sidestepping_left, _sidestepping_right, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| joystick | SDL_Joystick* | static | Current joystick handle; NULL if none open |
| joystick_active | int | global | Enable/disable joystick input at runtime |
| strafe_bounds | int[3] | static | Pulse modulation thresholds: {14000, 20000, 28000} |

## Key Functions / Methods

### enter_joystick
- Signature: `void enter_joystick(void)`
- Purpose: Initialize joystick subsystem and open first joystick device
- Inputs: None (reads `input_preferences->joystick_id`)
- Outputs/Return: None
- Side effects: Enables SDL joystick events, opens device handle (stored globally)
- Calls: `SDL_JoystickEventState(SDL_ENABLE)`, `SDL_JoystickOpen()`
- Notes: Gracefully handles failure (remains NULL); idempotent if joystick already open

### exit_joystick
- Signature: `void exit_joystick(void)`
- Purpose: Close joystick device and disable event delivery
- Inputs: None
- Outputs/Return: None
- Side effects: Closes SDL joystick handle, disables SDL joystick event state
- Calls: `SDL_JoystickClose()`
- Notes: Safe to call with NULL joystick

### joystick_buttons_become_keypresses
- Signature: `void joystick_buttons_become_keypresses(Uint8* ioKeyMap)`
- Purpose: Poll all active joystick buttons and write to keymap array
- Inputs: `ioKeyMap` ΓÇô keymap array to update (modified in-place)
- Outputs/Return: None
- Side effects: Updates `ioKeyMap[SDLK_BASE_JOYSTICK_BUTTON + i]` for each button state
- Calls: `SDL_JoystickNumButtons()`, `SDL_JoystickGetButton()`
- Notes: Supports up to 18 buttons (mapped to keycodes 73ΓÇô90); skips if joystick inactive

### process_joystick_axes
- Signature: `int process_joystick_axes(int flags, int tick)`
- Purpose: Poll all joystick axes, apply sensitivity/dead zones, and augment action flags with movement/view commands
- Inputs: 
  - `flags` ΓÇô existing action flags
  - `tick` ΓÇô frame counter for pulse modulation timing
- Outputs/Return: Updated action flags (augmented with strafe/yaw/pitch/velocity)
- Side effects: None
- Calls: `SDL_JoystickGetAxis()`, `mask_in_absolute_positioning_information()`
- Notes:
  - Maps 4 axes: strafe (0), forward velocity (1), yaw (2), pitch (3)
  - Applies sensitivity scaling: `value = sensitivity[i] * raw_axis`
  - Dead zones: clips values where `|value| < axis_bounds[i]` to 0
  - **Strafing uses pulse modulation** (not continuous): 
    - `abs_strafe < bounds[0]`: no motion
    - `bounds[0] Γëñ abs < bounds[1]`: 25% duty (fires on `tick % 4 == 0`)
    - `bounds[1] Γëñ abs < bounds[2]`: 50% duty (fires on `tick % 2 == 1`)
    - `bounds[2] Γëñ abs < bounds[3]`: 75% duty (fires on `tick % 4 != 0`)
    - `abs ΓëÑ bounds[2]`: continuous motion
  - Yaw/pitch/velocity use `mask_in_absolute_positioning_information()` for smooth analog support

## Control Flow Notes
- **Init**: `enter_joystick()` on engine startup
- **Per-frame update**: Both `joystick_buttons_become_keypresses()` and `process_joystick_axes()` called to poll hardware
- **Shutdown**: `exit_joystick()` on engine exit
- Input is polled, not event-driven (SDL joystick events are disabled)

## External Dependencies
- **SDL library**: `SDL.h` (SDL_Joystick, SDL_JoystickEventState, SDL_JoystickOpen/Close, SDL_JoystickGetButton/Axis, SDL_JoystickNumButtons)
- **player.h**: `mask_in_absolute_positioning_information()` (defined elsewhere; encodes yaw/pitch/velocity into action flags)
- **preferences.h**: global `input_preferences` pointer (holds axis mappings, sensitivities, bounds, joystick ID)
- **joystick.h**: header declaring this module's public interface
- **\<algorithm\>**: `std::max()` for button iteration
