# Source_Files/GameWorld/item_definitions.h

## File Purpose
Defines the item metadata table for the game engine. This header declares the structure for item properties (weapons, ammunition, powerups, keys, balls) and provides a static lookup table mapping item kinds to their game properties and resource identifiers.

## Core Responsibilities
- Define the `item_definition` structure for item metadata
- Declare a static global array of all item definitions indexed by item kind
- Specify weapon/ammo/powerup properties: singular/plural names, shape descriptors, carry limits, and environmental restrictions
- Serve as the authoritative reference for item capabilities used by game systems

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `item_definition` | struct | Stores metadata for a single item type: kind, name IDs, shape, carry limit, invalid environments |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `item_definitions[]` | array of `struct item_definition` | static (file scope, unless DONT_REPEAT_DEFINITIONS is set) | Lookup table for all ~45 item types; indexed by item kind to retrieve properties |

## Key Functions / Methods
None. This is a data-definition header only.

## Control Flow Notes
This file is purely declarative and used at initialization. The `item_definitions` array is referenced by other game systems (likely item spawning, inventory management, physics/collision, and UI rendering) to query item properties. The conditional `#ifndef DONT_REPEAT_DEFINITIONS` allows the structure to be included in multiple translation units or skipped in script-processing tools (e.g., Pfhortran).

## External Dependencies
- **Macros**: `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()` ΓÇö construct shape/collection identifiers (defined elsewhere)
- **Enum constants**: `_weapon`, `_ammunition`, `_powerup`, `_item`, `_ball` ΓÇö item kind types
- **Environment flags**: `_environment_vacuum`, `_environment_single_player` ΓÇö restrict items to certain world contexts
- **Magic constants**: `UNONE`, `NONE` ΓÇö likely "undefined" or "no shape" sentinels
- **Naming**: String table IDs (e.g., `0`, `1`, `2`) ΓÇö resolved at runtime to display names

**Not inferable from this file:**
- The definition of `shape_descriptor` type
- How `item_definitions` is actually indexed or iterated by other systems
- Behavior when `invalid_environments` flags prevent pickup
