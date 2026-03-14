# Source_Files/Lua/lua_mnemonics.h

## File Purpose
Defines string-to-integer mnemonic lookup tables for Lua scripting. Maps human-readable game element names (e.g., "knife", "rocket explosion", "kill monsters") to numeric constants used internally by the engine.

## Core Responsibilities
- Provide mnemonic mappings for game collections (creatures, items, weapons)
- Define enumerations for game states (completion, difficulty, monster actions)
- Supply effect, sound, and visual rendering parameters as named constants
- Enable Lua scripts to reference game entities by readable names instead of magic numbers
- Support multiple game systems: combat, inventory, level geometry, audio, visual effects

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `lang_def` | struct | Maps a string mnemonic (`name`) to a numeric value (`value`), terminated by null entry |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Collection_Mnemonics` | const `lang_def[]` | static | Maps entity class names (weapons, creatures, items) to IDs |
| `Lua_CompletionState_Mnemonics` | const `lang_def[]` | static | Level/objective completion states |
| `Lua_ControlPanelClass_Mnemonics` | const `lang_def[]` | static | Interactive panel types (rechargers, switches, terminals) |
| `Lua_DamageType_Mnemonics` | const `lang_def[]` | static | Damage source types (explosion, fire, melee, etc.) |
| `Lua_DifficultyType_Mnemonics` | const `lang_def[]` | static | Game difficulty levels |
| `Lua_EffectType_Mnemonics` | const `lang_def[]` | static | Visual/particle effects (explosions, splashes, teleports) |
| `Lua_FadeType_Mnemonics` | const `lang_def[]` | static | Screen transition effects |
| `Lua_GameType_Mnemonics` | const `lang_def[]` | static | Multiplayer game modes |
| `Lua_ScoringMode_Mnemonics` | const `lang_def[]` | static | Scoring systems for game modes |
| `Lua_ItemType_Mnemonics` | const `lang_def[]` | static | Pickupable items (weapons, ammo, powerups) |
| `Lua_MediaType_Mnemonics` | const `lang_def[]` | static | Environmental media types (water, lava, goo) |
| `Lua_MonsterClass_Mnemonics` | const `lang_def[]` | static | Enemy/creature categories (bitmask flags) |
| `Lua_MonsterAction_Mnemonics` | const `lang_def[]` | static | Monster behavior states |
| `Lua_MonsterMode_Mnemonics` | const `lang_def[]` | static | Monster targeting/awareness modes |
| `Lua_MonsterType_Mnemonics` | const `lang_def[]` | static | Specific monster variants (minor/major/specialized) |
| `Lua_OverlayColor_Mnemonics` | const `lang_def[]` | static | Radar/HUD overlay colors |
| `Lua_PlayerColor_Mnemonics` | const `lang_def[]` | static | Player team/color assignments |
| `Lua_PolygonType_Mnemonics` | const `lang_def[]` | static | Map geometry properties (triggers, platforms, teleporters) |
| `Lua_ProjectileType_Mnemonics` | const `lang_def[]` | static | Projectile/weapon variants |
| `Lua_SceneryType_Mnemonics` | const `lang_def[]` | static | Decorative/static objects in environment |
| `Lua_SideType_Mnemonics` | const `lang_def[]` | static | Wall texture mapping modes |
| `Lua_SizePref_Mnemonics` | const `lang_def[]` | static | UI element sizing preferences |
| `Lua_Sound_Mnemonics` | const `lang_def[]` | static | Audio effects (weapons, ambient, UI, creatures) |
| `Lua_TransferMode_Mnemonics` | const `lang_def[]` | static | Texture animation effects |
| `Lua_WeaponType_Mnemonics` | const `lang_def[]` | static | Player weapon types |
| `Lua_InventorySection_Mnemonics` | const `lang_def[]` | static | Inventory UI categories |
| `Lua_RendererType_Mnemonics` | const `lang_def[]` | static | Graphics rendering backend |
| `Lua_SensorBlipType_Mnemonics` | const `lang_def[]` | static | Motion tracker contact types |
| `Lua_FadeEffectType_Mnemonics` | const `lang_def[]` | static | Screen effect compositing modes |
| `Lua_TextureType_Mnemonics` | const `lang_def[]` | static | Texture rendering contexts |

## Key Functions / Methods
None. This is a data-only header file.

## Control Flow Notes
Not applicable. This file contains only static lookup table initialization, guarded by `#ifdef HAVE_LUA` to conditionally include Lua support.

## External Dependencies
- **`config.h`** ΓÇô Provides configuration symbols (e.g., `HAVE_LUA`)
- **`int32` type** ΓÇô Assumed defined in a broader headers (likely `<stdint.h>` or equivalent)
- **No function calls or dynamic dependencies** within this file
