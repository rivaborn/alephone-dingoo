# Source_Files/CSeries/cskeys.h

## File Purpose
Defines keyboard key code constants for input handling throughout the game engine. Provides platform-independent abstraction of keyboard scancodes and special keys (navigation, function keys).

## Core Responsibilities
- Define standard ASCII-equivalent key codes (DELETE, HOME, END, arrow keys)
- Define extended/Macintosh key codes with `kx` prefix (ESCAPE, HELP, FWD_DELETE, PAGE_UP/DOWN)
- Define function key constants (F1ΓÇôF15)
- Provide consistent naming for keyboard input mapping across the codebase

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This is a compile-time header for constant definitions. Used wherever keyboard input needs to be mapped to game actions. No runtime behavior.

## External Dependencies
- Only C preprocessor directives (`#ifndef`, `#define`, `#endif`)
- No external library dependencies

## Notes
- Three naming conventions present:
  - `kxNAME`: Extended/Macintosh scancodes (e.g., `kxESCAPE = 0x35`)
  - `kNAME`: ASCII-equivalent codes (e.g., `kDELETE = 0x08`)
  - `kcFN`: Function key codes (e.g., `kcF1 = 0x7A`)
- Comments indicate changes by Loren Petrich (Feb 2000) adding F13ΓÇôF15 support and renaming conventions
- Key values appear to be derived from Macintosh toolbox scancodes (per Inside Macintosh reference in comments)
