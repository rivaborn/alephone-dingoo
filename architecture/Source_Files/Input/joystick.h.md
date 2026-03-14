# Source_Files/Input/joystick.h

## File Purpose
Header file for joystick input handling in the Aleph One game engine. Provides interface to initialize joystick devices, translate button presses into keyboard events, and process analog axis inputs for use in the input system.

## Core Responsibilities
- Joystick device initialization and cleanup
- Translate joystick button presses into keyboard events for the keymap system
- Process analog joystick axis data (sticks, triggers)
- Define constants for mapping joystick buttons to key indices in the global keymap array

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| SDLK_BASE_JOYSTICK_BUTTON | #define (int) | global | Start index (73) for joystick button keycodes in the keymap array; positioned after mouse buttons and before key 91 |
| NUM_SDL_JOYSTICK_BUTTONS | #define (int) | global | Maximum number of joystick buttons supported (18) |

## Key Functions / Methods

### enter_joystick
- Signature: `void enter_joystick(void)`
- Purpose: Initialize joystick device(s) and prepare for input polling
- Inputs: None
- Outputs/Return: None
- Side effects: Likely opens SDL joystick devices, allocates state
- Calls: Not visible in this file

### exit_joystick
- Signature: `void exit_joystick(void)`
- Purpose: Clean up joystick device(s) and release resources
- Inputs: None
- Outputs/Return: None
- Side effects: Likely closes SDL joystick devices, deallocates state
- Calls: Not visible in this file

### joystick_buttons_become_keypresses
- Signature: `void joystick_buttons_become_keypresses(Uint8* ioKeyMap)`
- Purpose: Poll joystick button state and update the keymap array with pressed buttons as key events
- Inputs: `ioKeyMap` ΓÇô pointer to global key state array (SDL keymap)
- Outputs/Return: None (modifies `ioKeyMap` in-place)
- Side effects: Updates global keymap with current button state
- Calls: Not visible in this file

### process_joystick_axes
- Signature: `int process_joystick_axes(int flags, int tick)`
- Purpose: Poll joystick analog axes and integrate axis movements into game input
- Inputs: `flags` ΓÇô control flags (purpose not inferable); `tick` ΓÇô frame/time identifier
- Outputs/Return: Integer result (purpose not inferable from signature)
- Side effects: Likely updates game input state or returns input delta
- Calls: Not visible in this file

## Control Flow Notes
Lifecycle: `enter_joystick()` called during engine init; `joystick_buttons_become_keypresses()` and `process_joystick_axes()` polled each frame in the input update phase; `exit_joystick()` called during shutdown. Buttons are translated to key events and mixed into the global keymap, while axes are processed separately (possibly as relative motion or input commands).

## External Dependencies
- **SDL (Simple DirectMedia Layer)** ΓÇô SDLK_* constants and `Uint8` type indicate SDL integration for cross-platform joystick support
- Global keymap array ΓÇô defined elsewhere; modified by `joystick_buttons_become_keypresses()`
- Aleph One game engine runtime ΓÇô initialization/shutdown hooks
