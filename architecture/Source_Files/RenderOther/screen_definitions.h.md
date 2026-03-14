# Source_Files/RenderOther/screen_definitions.h

## File Purpose
Centralized enum defining base resource IDs for game screen types (intro, menu, prologue, credits, etc.). Used to map screen types to PICT image resource ranges across 8-bit, 16-bit, and 32-bit color depths.

## Core Responsibilities
- Define base resource ID constants for each screen type
- Provide offset mapping for bitdepth variants (8-bit + 0, 16-bit + 10000, 32-bit + 20000)
- Establish a naming convention for screen resource organization

## Key Types / Data Structures
None (header constants only).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `INTRO_SCREEN_BASE` | `int` (enum constant) | global | Base resource ID for intro sequence (1000) |
| `MAIN_MENU_BASE` | `int` (enum constant) | global | Base resource ID for main menu (1100) |
| `PROLOGUE_SCREEN_BASE` | `int` (enum constant) | global | Base resource ID for prologue (1200) |
| `EPILOGUE_SCREEN_BASE` | `int` (enum constant) | global | Base resource ID for epilogue (1300) |
| `CREDIT_SCREEN_BASE` | `int` (enum constant) | global | Base resource ID for credits (1400) |
| `CHAPTER_SCREEN_BASE` | `int` (enum constant) | global | Base resource ID for chapter interludes (1500) |
| `COMPUTER_INTERFACE_BASE` | `int` (enum constant) | global | Base resource ID for computer terminals/UI (1600) |
| `INTERFACE_PANEL_BASE` | `int` (enum constant) | global | Base resource ID for HUD panels (1700) |
| `FINAL_SCREEN_BASE` | `int` (enum constant) | global | Base resource ID for endgame/final screen (1800) |

## Key Functions / Methods
None.

## Control Flow Notes
Pure data definition. Consumed by rendering/resource-loading code to map screen types to their corresponding PICT image resources. The offset scheme (8-bit base, 16-bit +10000, 32-bit +20000) suggests a resource management strategy supporting multiple color depths.

## External Dependencies
None.
