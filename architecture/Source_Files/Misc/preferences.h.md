# Source_Files/Misc/preferences.h

## File Purpose
Defines data structures for storing and managing game preferences across graphics, networking, input, player settings, and environment configuration. Declares initialization and I/O functions for the preferences system. Serves as the central configuration schema for the Aleph One game engine.

## Core Responsibilities
- Define serializable data structures for graphics, sound, network, player, input, and environment preferences
- Declare preference initialization and lifecycle functions (read/write/handle)
- Provide extern declarations for global preference data instances
- Define enumerations for input modifiers, shell keys, joystick mappings, and mouse actions
- Aggregate preferences from multiple subsystems (OpenGL, chase cam, crosshairs, sound manager)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `graphics_preferences_data` | struct | Screen mode, OpenGL config, alpha blending, CPU usage, DirectX backend flag |
| `serial_number_data` | struct | License/serial validation: network-only flag, serial number, user name, tokenized serial |
| `network_preferences_data` | struct | Network game config: game type, difficulty, port, protocol, audio codec, metaserver login, chat settings |
| `player_preferences_data` | struct | Player state: name, color, team, difficulty, music toggle, chase cam & crosshair data |
| `input_preferences_data` | struct | Input device config: keycodes, mouse sensitivity, joystick mappings, button actions, input modifiers |
| `environment_preferences_data` | struct | Level environment: map/physics/shapes/sounds files with checksums, theme dir, Lua script settings |
| `OGL_ConfigureData` | struct (from OGL_Setup.h) | OpenGL rendering: texture config per type, fog, Z-buffer, void color, landscape colors, anisotropy |
| `ChaseCamData` | struct (from ChaseCam.h) | Third-person camera: position offsets, damping, spring, opacity, activation flags |
| `CrosshairData` | struct (from Crosshairs.h) | Crosshair rendering: color, thickness, opacity, shape (crosshairs vs circle) |
| `SoundManager::Parameters` | struct (from SoundManager.h) | Audio config: channel count, volume, sample rate, music volume, netmic settings |
| Input modifier flags enum | enum | `_inputmod_interchange_run_walk`, `_inputmod_dont_switch_to_new_weapon`, `_inputmod_invert_mouse`, `_inputmod_dont_auto_recenter`, etc. |
| Shell key binding enum | enum | `_key_inventory_left/right`, `_key_switch_view`, `_key_zoom_in/out`, `_key_toggle_fps`, `_key_activate_console` |
| Joystick mapping enum | enum | `_joystick_strafe`, `_joystick_velocity`, `_joystick_yaw`, `_joystick_pitch` |
| Mouse button action enum | enum | `_mouse_button_does_nothing`, `_mouse_button_fires_left_trigger`, `_mouse_button_fires_right_trigger` |
| Network protocol enum | enum | `_network_game_protocol_ring`, `_network_game_protocol_star` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `graphics_preferences` | `graphics_preferences_data*` | extern | Global graphics settings instance |
| `serial_preferences` | `serial_number_data*` | extern | Global license/serial data instance |
| `network_preferences` | `network_preferences_data*` | extern | Global network game settings instance |
| `player_preferences` | `player_preferences_data*` | extern | Global player settings instance |
| `input_preferences` | `input_preferences_data*` | extern | Global input configuration instance |
| `sound_preferences` | `SoundManager::Parameters*` | extern | Global audio parameters instance |
| `environment_preferences` | `environment_preferences_data*` | extern | Global environment/map settings instance |
| `DEFAULT_MONITOR_REFRESH_FREQUENCY` | const float (60) | global | Monitor refresh rate constant |

## Key Functions / Methods

### initialize_preferences
- **Signature:** `void initialize_preferences(void);`
- **Purpose:** Initialize the preferences system at engine startup; allocate/setup preference data structures.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Allocates global preference data structures; may set defaults.
- **Calls:** Defined elsewhere (likely preferences.c).
- **Notes:** Called early in engine initialization; must run before game state setup.

### read_preferences
- **Signature:** `void read_preferences();`
- **Purpose:** Load saved preferences from persistent storage (file system).
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Populates all global preference pointers with saved data; I/O to file system.
- **Calls:** Defined elsewhere.
- **Notes:** Typically called during game startup after `initialize_preferences()`.

### handle_preferences
- **Signature:** `void handle_preferences(void);`
- **Purpose:** Display and manage the preferences/settings dialog; allow user to modify settings.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Modifies global preference structures based on user input; may update UI state.
- **Calls:** Defined elsewhere (likely dialog/UI code).
- **Notes:** Called in response to user selecting preferences menu.

### write_preferences
- **Signature:** `void write_preferences(void);`
- **Purpose:** Persist current preference settings to storage.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Writes all global preference structures to disk; I/O to file system.
- **Calls:** Defined elsewhere.
- **Notes:** Called at game shutdown or after preferences changes.

## Control Flow Notes
This file defines the **configuration/preferences subsystem**, which is orthogonal to the main game frame loop:
- **Initialization:** `initialize_preferences()` runs once at engine startup.
- **Load time:** `read_preferences()` called during game launch to restore user settings.
- **Runtime:** `handle_preferences()` called asynchronously when user opens the preferences dialog (pauses or blocking).
- **Shutdown:** `write_preferences()` called at game exit to persist changes.

The preferences data structures are populated once and referenced throughout the engine's lifetime (graphics, audio, input, network, rendering subsystems all consume these globals).

## External Dependencies
- **interface.h:** Core engine constants, prototypes, and game state enums.
- **ChaseCam.h:** Chase camera data structure and configuration functions.
- **Crosshairs.h:** Crosshair data structure and rendering interface.
- **OGL_Setup.h:** OpenGL configuration structure and OpenGL subsystem functions.
- **shell.h:** Screen mode data structure and shell-level constants (keys, input devices).
- **SoundManager.h:** Sound manager parameters structure and audio subsystem interface.

---

**Notes:** This is a configuration/data-definition header with minimal logic. The actual preference I/O, validation, and dialog handling are implemented in corresponding `.c` or `.cpp` files (e.g., `preferences.c`). The file aggregates heterogeneous preference types from multiple subsystems into a cohesive, serializable structure.
