# Source_Files/Lua/language_definition.h

## File Purpose
Lua language binding definitions that map human-readable symbolic names to numeric constants for game entities. Provides dual naming conventions (underscore-prefixed and plain) for backwards compatibility. Used to allow Lua scripts to reference items, monsters, sounds, and game mechanics by name rather than raw hex values.

## Core Responsibilities
- Define symbolic mnemonics for inventory items (weapons, magazines, powerups)
- Define symbolic mnemonics for enemy/NPC types and variants
- Define symbolic mnemonics for damage sources
- Define symbolic mnemonics for monster classifications and behavioral states
- Define symbolic mnemonics for player input actions and monster actions
- Define symbolic mnemonics for visual effects (fades and screen tints)
- Define symbolic mnemonics for sound effect IDs
- Define symbolic mnemonics for projectile/weapon fire types
- Define symbolic mnemonics for polygon/terrain properties and triggers
- Define symbolic mnemonics for game modes (CTF, KOTH, etc.)
- Provide backwards-compatible dual naming (with/without `_` prefix)

## Key Types / Data Structures
None

## Global / File-Static State
None

## Key Functions / Methods
None

## Control Flow Notes
This file is a pure data structureΓÇöa lookup table intended to be included inline within a larger Lua binding array or table. Compilation is gated by `#ifdef HAVE_LUA` (config.h). The structure suggests it's meant to be embedded as initializer data in a host function or global array.

## External Dependencies
- `#include "config.h"` ΓÇö provides `HAVE_LUA` preprocessor conditional
- References undefined symbolic constants (defined elsewhere):
  - `_game_of_most_points`, `_game_of_most_time`, etc. (game scoring modes)
  - `_panel_is_oxygen_refuel`, `_panel_is_shield_refuel`, etc. (panel types)
  - `_weapon_fist`, `_weapon_pistol`, etc. (weapon identifiers)

## Notes
- **Versioning:** File contains legacy names retained for backwards compatibility (e.g., comment at lines ~8ΓÇô10 noting "shotgun" dual definitions fixed in later versions)
- **Comment on usage:** "jkvw: I'm not sure how this is intended to be used, so I'm leaving it alone" (default_camera entry)
- **Hex constant consistency:** Uses 0x prefix throughout; values range 0x00ΓÇô0x26 (items/monsters), 0x00ΓÇô0x17 (damage), 0x0001ΓÇô0x8000 (monster classes, power-of-two bit flags), 0ΓÇô214 (sounds)
- **Sound namespace density:** 215 sound entries (0ΓÇô214), suggesting a comprehensive audio palette for the engine
