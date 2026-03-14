# Source_Files/Misc/interface_menus.h

## File Purpose
Defines menu and menu-item constants for the Aleph One game engine. Provides enumeration IDs for the game menu (pause, save, quit) and interface menu (new game, load, preferences, etc.), and optionally specifies menu names for macOS NIB-based UI initialization.

## Core Responsibilities
- Define game-context menu constants (`mGame` and associated item IDs)
- Define interface menu constants (`mInterface` and associated item IDs)
- Define a placeholder/fake menu to suppress empty menu bar display at exit
- Store menu names for macOS NIB UI framework when `USES_NIBS` is enabled
- Map between menu resource IDs (128, 129, 130) and semantic names

## Key Types / Data Structures
None. File contains only enums and constants.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `NumMenus` | `const int` | Global | Count of menus (3) when USES_NIBS enabled |
| `MenuNames` | `const CFStringRef[]` | Global | Array of menu resource names for macOS NIB loader; order is reversed (right-to-left insertion) |

## Key Functions / Methods
None. This is a header-only definitions file.

## Control Flow Notes
This file is part of the menu initialization layer, used during engine startup on macOS. The comment indicates menus are inserted right-to-left in `interface_macintosh.cpp` to achieve the desired final order (Game, Interface, Fake_Empty). The `USES_NIBS` conditional suggests legacy Mac support for Interface Builder NIB files; when enabled, menu names are provided for dynamic resource loading.

## External Dependencies
- Core Foundation: `CFStringRef`, `CFSTR()` macro (macOS-specific)
- Referenced in: `interface_macintosh.cpp` (per comments)
- Consumed by: Menu bar initialization and event dispatching during game runtime

## Notes
- Menu IDs are sequential (128, 129, 130), suggesting they map to resource IDs in a resource file
- Menu item IDs start at 1 within each menu enum, suggesting 1-indexed menu item dispatch
- The "Fake_Empty" menu exists purely to hide an empty slot in the menu bar on exit, a macOS-specific UI polish detail
