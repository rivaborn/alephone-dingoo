# Source_Files/Misc/preferences_private.h

## File Purpose
Private header for the preferences system containing UI dialog item identifier constants and enums. Defines the structure and layout of preference panes (Graphics, Player, Sound, Input, Environment) with item indices for each control within them. Used by preferences implementers, not the broader game engine.

## Core Responsibilities
- Define dialog item identifier enums for each preferences pane
- Map preference pane categories to MacOS type codes (OSType signatures)
- Provide item indices for UI controls within each preference pane
- Organize preferences into logical groups with a group registry

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ditlGRAPHICS | enum | Dialog item IDs for graphics preference pane (monitor, acceleration, color depth, resolution, detail, brightness, OpenGL options) |
| ditlPLAYER | enum | Dialog item IDs for player preference pane (difficulty, name, color, team, camera, crosshairs) |
| ditlSOUND | enum | Dialog item IDs for sound preference pane (stereo, panning, ambient, quality, volume, music) |
| ditlINPUT | enum | Dialog item IDs for input preference pane (mouse/keyboard/InputSprocket control, sensitivity, inversion, weapon switching) |
| ditlENVIRONMENT | enum | Dialog item IDs for environment preference pane (map, physics, shapes, sounds, patches) |
| strPREFERENCES_GROUPS | enum | Group registry mapping group indices to preference categories |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Window_Preferences | CFStringRef | global (conditional) | MacOS window resource name for preferences dialog |
| iTABS | int | global (conditional) | Number of preference pane tabs |
| Sig_Pane, Sig_Graphics, Sig_Sound, Sig_Player, Sig_Input, Sig_Environment | OSType | global (conditional) | MacOS type codes for marking panes and their components |

## Key Functions / Methods
None. This file contains only constants and enums.

## Control Flow Notes
Part of preferences initialization/configuration flow. Preferences are set up before the main game loop. The enum values (starting at 4001ΓÇô4005) are dialog item resource IDs referenced during preferences dialog construction. The group registry (strPREFERENCES_GROUPS) suggests preferences are loaded/saved grouped by category.

## External Dependencies
- **MacOS-specific**: `#ifdef USES_NIBS` gates MacOS Carbon/Cocoa-related constants (CFStringRef, OSType)
- **Related files** (from comments): preferences_macintosh.cpp (original source of dialog constants), preferences system implementation
- **Notes**: Enum ordinals (4001ΓÇô4005) are likely resource IDs in a .rsrc or .nib file on MacOS
