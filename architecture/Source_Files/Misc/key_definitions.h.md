# Source_Files/Misc/key_definitions.h

## File Purpose
Defines keyboard mapping configurations for the game engine, supporting multiple input layouts (standard, left-handed, PowerBook) and platform-specific key definitions (SDL and native Mac/PC). Maps physical keys to in-game actions like movement, triggers, and weapon cycling.

## Core Responsibilities
- Define data structures for key mappings (`key_definition`, `blacklist_data`, `special_flag_data`)
- Provide three predefined key configuration arrays with different physical layouts
- Support platform-specific key encoding (SDL keycodes vs. native Mac scancodes/hex values)
- Support device-specific layouts (Dingoo handheld with alternate mappings)
- Maintain a centralized registry of all available key setups
- Expose the currently active key configuration to input handling code

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `blacklist_data` | struct | Tracks key combinations that should be ignored/disabled (2 key offsets + masks) |
| `special_flag_data` | struct | Defines special flag behaviors (type, flag, alternate, persistence duration) |
| `key_definition` | struct | Single key-to-action mapping; platform-dependent (SDL vs offset/mask) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `standard_key_definitions` | `key_definition[]` | static | Default WASD-like layout; uses keypad (KP8/5/4/6) or arrow keys depending on platform |
| `left_handed_key_definitions` | `key_definition[]` | static | Left-handed layout using arrow keys for movement, semicolon/quote for weapon cycling |
| `powerbook_key_definitions` | `key_definition[]` | static | Laptop-optimized layout (O/L/K/; for movement) for PowerBook keyboards |
| `all_key_definitions` | `key_definition*[]` | static | Array of pointers to all 3 layouts, indexed by setup ID |
| `current_key_definitions` | `key_definition[]` | extern | Active key configuration; linked at runtime (defined elsewhere) |

## Key Functions / Methods
None. This is a pure data definition header.

**Trivial helpers:** Macros `NUMBER_OF_STANDARD_KEY_DEFINITIONS`, `NUMBER_OF_LEFT_HANDED_KEY_DEFINITIONS`, `NUMBER_OF_POWERBOOK_KEY_DEFINITIONS` compute array sizes using `sizeof()`.

## Control Flow Notes
This header is included by `vbl.c` and `vbl_macintosh.c` (vertical blank interrupt handlers), likely during the frame update loop when processing keyboard input. The current active layout (`current_key_definitions`) is selected at runtime and used to translate raw key events to game actions.

## External Dependencies
- **SDL (conditional):** `SDLKey` type for cross-platform key handling; prefixed with `SDLK_` (e.g., `SDLK_UP`, `SDLK_SPACE`)
- **Platform conditionals:** `#ifdef SDL` and `#ifdef HAVE_DINGOO` for device-specific mappings
- **Undefined action flags:** `_moving_forward`, `_left_trigger_state`, `_toggle_map`, etc. (defined elsewhere; likely in an enum or header)
- **Integer types:** `int16`, `int32`, `uint32` (defined elsewhere, likely stdint or platform-specific types)
