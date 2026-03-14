# Source_Files/shell.h

## File Purpose
Core header defining the shell/main interface layer for the game engine. Declares initialization functions, screen mode configuration, input device enumeration, and protocols for shape rendering, color management, and XML-based cheat parsing.

## Core Responsibilities
- Define display configuration structure (`screen_mode_data`) and input device modes
- Declare game initialization entry points (shape handler, MML scripts, preferences)
- Export shape rendering interface with support for RLE encoding, illumination, and colorization
- Provide color lookup functions for player and UI elements
- Manage game window updates and on-screen text output
- Support XML-based configuration (cheat codes via MML)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `screen_mode_data` | struct | Display configuration (resolution, bit depth, acceleration, HUD scale, gamma) |
| Input device enum | enum | Enumeration of input modes (keyboard/gamepad, mouse yaw/pitch/velocity, Input Sprocket) |
| Resource IDs enum | enum | String resource IDs for UI prompts (save game, load replay, etc.) |

## Global / File-Static State
None.

## Key Functions / Methods

### global_idle_proc
- Signature: `void global_idle_proc(void);`
- Purpose: Main idle/frame loop entry point for the game engine.
- Inputs: None
- Outputs/Return: None
- Side effects: Likely updates game state, rendering, and input processing each frame
- Calls: Not visible in this file
- Notes: Likely called repeatedly from the main application loop

### Cheats_GetParser
- Signature: `XML_ElementParser *Cheats_GetParser();`
- Purpose: Returns an XML element parser for parsing cheat code definitions from MML.
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser configured for cheats
- Side effects: None (parser returned, not created each call; likely cached)
- Calls: Not visible in this file
- Notes: Integrates with XML configuration system for cheat definitions

### LoadBaseMMLScripts
- Signature: `void LoadBaseMMLScripts();`
- Purpose: Load base Marathon Markup Language scripts during initialization.
- Inputs: None
- Outputs/Return: None
- Side effects: Loads and parses MML configuration files; modifies game state
- Calls: Not visible in this file
- Notes: Called during engine startup; foundation for all MML-based configuration

### initialize_shape_handler
- Signature: `void initialize_shape_handler(void);`
- Purpose: Initialize the shape (graphics resource) handling subsystem.
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up shape caching, resource loading, memory allocation
- Calls: Not visible in this file
- Notes: Must be called before any shape rendering

### get_shape_surface
- Signature: `SDL_Surface *get_shape_surface(int shape, int collection = NONE, byte** outPointerToPixelData = NULL, float inIllumination = -1.0f, bool inShrinkImage = false);`
- Purpose: Retrieve an SDL surface for a shape with optional RLE decompression, illumination shading, team colorization, and quarter-size shrinking.
- Inputs: 
  - `shape`: Low-level shape index or encoded shape/collection/CLUT info
  - `collection`: Collection ID; if `NONE`, `shape` encodes all three elements
  - `outPointerToPixelData`: Output pointer for RLE pixel data (caller must free)
  - `inIllumination`: 0.0ΓÇô1.0 for darkening; < 0 uses collection CLUT directly
  - `inShrinkImage`: Nearest-neighbor scale to 1/4 size (RLE shapes only)
- Outputs/Return: SDL_Surface pointer; RLE data pointer written to `outPointerToPixelData` if applicable
- Side effects: Allocates SDL_Surface and potentially RLE decompression buffer
- Calls: Not visible in this file
- Notes: Supports colorization for team/player colors when illumination >= 0; RLE shrinking is lossy (no smoothing); caller responsible for freeing RLE buffer

### load_environment_from_preferences
- Signature: `void load_environment_from_preferences(void);`
- Purpose: Load saved preferences (video, audio, gameplay settings) during startup.
- Inputs: None
- Outputs/Return: None
- Side effects: Reads preference file; updates game configuration (screen mode, input settings, etc.)
- Calls: Not visible in this file
- Notes: Called during initialization after engine core is ready

### update_game_window
- Signature: `void update_game_window(void);`
- Purpose: Update the game window display (render current frame).
- Inputs: None
- Outputs/Return: None
- Side effects: Redraws window content; may swap buffers
- Calls: Not visible in this file
- Notes: Likely called once per frame during the main loop

## Control Flow Notes
This header represents the initialization and frame-loop interface:
- **Startup**: `LoadBaseMMLScripts()` ΓåÆ `initialize_shape_handler()` ΓåÆ `load_environment_from_preferences()`
- **Frame loop**: `global_idle_proc()` called repeatedly; `update_game_window()` likely called at end of each frame
- Color/shape functions called on-demand during rendering
- XML cheat parser obtained once and reused for all cheat definitions

## External Dependencies
- **Forward declared**: `FileSpecifier` (defined elsewhere)
- **Included**: `XML_ElementParser.h`
- **Used but not defined here**: `RGBColor`, `SDL_Color`, `SDL_Surface`, `byte`
- **Resource/constant definitions**: String resource IDs from string table
