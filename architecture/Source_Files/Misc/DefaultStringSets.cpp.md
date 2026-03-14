# Source_Files/Misc/DefaultStringSets.cpp

## File Purpose
Provides compiled-in default string sets for the Aleph One game engine, replacing MacOS resource files. Defines UI text, error messages, game statistics labels, weapon/item names, and difficulty levels that are registered at static initialization time.

## Core Responsibilities
- Defines string arrays for error messages, menu labels, and game UI prompts
- Initializes network game type, difficulty level, and game mode descriptions
- Registers team color names and in-game item/weapon name references
- Provides default strings for weapons, items, and game statistics tracking
- Implements compile-time string-set registration via static object instantiation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `AutoStringSetBuilder` | class | Utility for registering string sets during static initialization by calling `TS_PutCString()` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sStringSetNumber128` | `const char*[]` | static | Error messages (68 entries) |
| `sStringSetNumber129` | `const char*[]` | static | Filenames (Shapes, Sounds, Maps, etc.) |
| `sStringSetNumber130` | `const char*[]` | static | Top-level UI menu items |
| `sStringSetNumber131` | `const char*[]` | static | File save/load prompts |
| `sStringSetNumber132` | `const char*[]` | static | Network error messages |
| `sStringSetNumber133` | `const char*[]` | static | Keyboard key code names |
| `sStringSetNumber134` | `const char*[]` | static | Preferences advice/hints |
| `sStringSetNumber135` | `const char*[]` | static | Computer interface display strings |
| `sStringSetNumber136` | `const char*[]` | static | Join dialog messages |
| `sStringSetNumber137` | `const char*[]` | static | Weapon names (10 entries) |
| `sStringSetNumber138` | `const char*[]` | static | File search paths |
| `sStringSetNumber139` | `const char*[]` | static | Preferences category groupings |
| `sStringSetNumber140` | `const char*[]` | static | Post-game network statistics labels |
| `sStringSetNumber141` | `const char*[]` | static | Network game setup labels |
| `sStringSetNumber142` | `const char*[]` | static | New join dialog messages |
| `sStringSetNumber143` | `const char*[]` | static | Network progress status strings |
| `sStringSetNumber150` | `const char*[]` | static | Item names (43 entries) |
| `sStringSetNumber151` | `const char*[]` | static | Item type categories |
| `sStringSetNumber153` | `const char*[]` | static | Network statistics strings (kills, deaths, etc.) |
| `sStringSetNumber200` | `const char*[]` | static | OpenGL color-picker prompts |
| `sDifficultyLevelsStrings` | `const char*[]` | static | Game difficulty options (5 entries) |
| `sNetworkGameTypesStrings` | `const char*[]` | static | Network game modes (8 entries) |
| `sEndConditionTypeStrings` | `const char*[]` | static | Game end conditions (3 entries) |
| `sSingleOrNetworkStrings` | `const char*[]` | static | Game type selector (2 entries) |
| `sTeamColorNamesStrings` | `const char*[]` | static | Team color names (8 entries) |

## Key Functions / Methods

### AutoStringSetBuilder::AutoStringSetBuilder
- **Signature:** `AutoStringSetBuilder(short inStringSetID, const char** inStrings, short inNumStrings)`
- **Purpose:** Register a complete string set in the global text string repository during static initialization
- **Inputs:** String set ID (e.g., 128), pointer to array of C-strings, count of strings
- **Outputs/Return:** None (constructor)
- **Side effects:** Calls `TS_PutCString()` for each string, populating the global string registry
- **Calls:** `TS_PutCString()`
- **Notes:** Exploits C++ static object initialization to avoid explicit runtime registration code; no heap storage needed as objects are never destroyed

## Control Flow Notes
This file executes entirely at static initialization time (before `main()`). The `BUILD_STRINGSET` macro expands to create static `AutoStringSetBuilder` instances, each triggering the constructor which registers all strings in a set. By the time the engine's runtime begins, all 16 string sets are pre-populated in the central text-string repository.

The design allows the engine to function without external MML (MapML) resource files while still supporting MML-provided strings at runtime (which would override these defaults).

## External Dependencies
- **Includes:** `config.h` (build config), `cseries.h` (platform abstraction), `TextStrings.h` (string repository API), `network_dialogs.h` (network dialog string IDs), `player.h` (team colors string ID)
- **External symbols used:** `TS_PutCString()` (defined elsewhere; registers a single string), `kDifficultyLevelsStringSetID`, `kNetworkGameTypesStringSetID`, `kEndConditionTypeStringSetID`, `kSingleOrNetworkStringSetID`, `kTeamColorsStringSetID` (macro constants for string set IDs)
