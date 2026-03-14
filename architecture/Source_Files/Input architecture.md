# Subsystem Overview

## Purpose
The Input subsystem abstracts hardware device input (keyboards, mice, joysticks) and converts device state into game action flags and movement deltas (yaw, pitch, velocity). It provides a cross-platform interface for initializing input devices, polling them each frame, and applying user preference settings (sensitivity, acceleration, dead zones, button mappings) to produce deterministic game commands.

## Key Files
| File | Role |
|------|------|
| joystick_sdl.cpp | SDL joystick polling, button-to-keypress mapping, analog axis scaling with dead zones and pulse-modulation for strafing |
| joystick.h | Public interface for joystick lifecycle (init/shutdown) and keymap conversion |
| mouse_sdl.cpp | SDL mouse movement capture, delta normalization with sensitivity/acceleration, scroll wheel-to-weapon-cycling mapping, mouse button simulation as keycodes |
| mouse.h | Public interface for mouse lifecycle, per-frame polling, and SDL keypress simulation |
| ISp_Support.cpp | macOS InputSprocket framework integration; 44 action definitions (21 global networked, 23 local), device polling, refractory period enforcement on local events |
| ISp_Support.h | InputSprocket lifecycle (init/shutdown/start/stop) and control configuration queries |

## Core Responsibilities
- Initialize and shutdown input device subsystems during game startup and shutdown
- Poll joystick buttons each frame and translate them to virtual keycodes via user keymap
- Poll joystick analog axes (strafe, velocity, yaw, pitch) and scale outputs via sensitivity/dead zones
- Implement pulse-modulation for strafing to enforce discrete intensity bands (avoiding smooth analog strafing)
- Capture mouse movement deltas, apply sensitivity and acceleration curves, and convert to game yaw/pitch/velocity actions
- Map mouse buttons to weapon fire triggers and scroll wheel to weapon cycling
- Apply user preference settings (sensitivity multipliers, axis inversion, acceleration, button mappings, device selection) to all input
- Maintain thread-safe mouse state synchronization on platforms using Carbon Event Manager

## Key Interfaces & Data Flow
**Exposes to game engine:**
- `enter_joystick()` / `exit_joystick()` ΓÇö joystick lifecycle
- `joystick_buttons_become_keypresses()` ΓÇö map joystick buttons to SDL keycodes in global keymap
- `enter_mouse()` / `exit_mouse()` ΓÇö mouse lifecycle
- `test_mouse()` ΓÇö poll and return normalized mouse state (yaw, pitch, velocity deltas)
- `recenter_mouse()` ΓÇö warp cursor to screen center for relative tracking
- `mouse_buttons_become_keypresses()` ΓÇö map mouse buttons to SDL keycodes
- `mouse_scroll()` ΓÇö handle scroll wheel events

**Consumes from other subsystems:**
- `preferences.h` ΓÇö global `input_preferences` struct containing axis mappings, sensitivities, dead zones, button bindings, modifier flags, device selection
- `player.h` ΓÇö action flag enums and `mask_in_absolute_positioning_information()` for encoding yaw/pitch/velocity into action flags; `get_absolute_pitch_range()` for pitch clamping
- `world.h`, `map.h` ΓÇö game state structures and coordinate type definitions
- `shell.h` ΓÇö input device type enums, `get_game_state()`, `FrontWindow()` for input context
- **SDL library** ΓÇö joystick enumeration, button state, analog axis values, mouse position and button state
- **InputSprocket (macOS)** ΓÇö device enumeration, virtual element definitions, button state polling

## Runtime Role
- **Initialization:** `enter_joystick()` and `enter_mouse()` called during game startup to discover devices and register input handlers
- **Per-frame:** Input polling routines called in main game loop to sample joystick buttons/axes and mouse deltas; raw input converted to action flags and movement deltas with user preference scaling applied
- **Shutdown:** `exit_joystick()` and `exit_mouse()` called during game shutdown to release resources and disable event handlers

## Notable Implementation Details
- Joystick strafing uses pulse-modulation: analog axis values divided into 3 discrete intensity bands, each controlling a duty cycle (frequency of keypress simulation) to avoid smooth analog strafing
- Mouse input on macOS uses Carbon Event Manager with critical regions for thread-safe state synchronization between event handler thread and input polling thread
- Mouse movement normalization applies per-axis sensitivity multipliers and optional acceleration curves using fixed-point arithmetic (`_fixed` type, `FIXED_FRACTIONAL_BITS`)
- Joystick axes subject to dead-zone bounds clipping and sensitivity scaling via `input_preferences` before conversion to action flags
- InputSprocket implementation enforces refractory periods on local-event buttons to prevent rapid repeat triggering of actions like pause/quit
- ISp disabled under Carbon API compatibility mode; SDL implementations provide cross-platform fallback
