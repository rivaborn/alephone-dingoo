# Source_Files/Misc/preferences_widgets_sdl.cpp

## File Purpose
Implements SDL-specific GUI widgets for the preferences system, enabling users to select environment assets (themes, maps, physics, sounds) and preview crosshairs. Provides the dialog-based file browser and visual preview components for game configuration.

## Core Responsibilities
- Implements `w_env_select` widget for browsing and selecting environment files (themes, maps, physics data, shapes, sounds)
- Implements `w_crosshair_display` widget for rendering a crosshair preview in the preferences UI
- Searches asset directories using FileFinder and organizes files with hierarchical structure (by directory)
- Creates and manages modal dialogs for file selection with formatted item lists
- Handles callbacks when users make selections (via `selection_made_callback_t`)
- Manages SDL surface lifecycle for crosshair preview rendering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `w_env_select` | class (widget subclass) | Button widget opening a modal file-selection dialog |
| `w_crosshair_display` | class (widget subclass) | Non-interactive preview widget rendering a crosshair bitmap |
| `env_item` | struct | Represents a file or directory in the selection list with name, specifier, indent level, and selectability flag |
| `w_env_list` | class template (w_list<env_item>) | Scrollable list widget with selectable/unselectable items and theme-aware rendering |
| `FindThemes` | class (FileFinder subclass) | Searches for theme.mml files in asset directories |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | external | Paths to search for asset files (defined in shell_sdl.cpp) |
| `local_data_dir` | `DirectorySpecifier` | external (macOS only) | Platform-specific local data directory |

## Key Functions / Methods

### w_env_select::select_item
- Signature: `void select_item(dialog *parent)`
- Purpose: Open a modal dialog allowing the user to browse and select an environment file (theme, map, physics, shapes, or sounds)
- Inputs: Parent dialog pointer
- Outputs/Return: None (modifies internal `item` and calls callback)
- Side effects: Creates/runs modal dialog, clears screen, may invoke `mCallback` on selection
- Calls: `FindThemes::Find()` or `FindAllFiles::Find()`, `FileSpecifier::SplitPath()`, dialog creation (`d.run()`)
- Notes: Hierarchically organizes files by directory with indentation; distinguishes themes (found by theme2.mml) from other assets (found by type code)

### w_env_select::select_item_callback
- Signature: `static void select_item_callback(void *arg)`
- Purpose: Static trampoline callback for SDL widget framework to invoke `select_item()`
- Inputs: `arg` (cast to `w_env_select*`)
- Outputs/Return: None
- Side effects: Forwards to instance method
- Calls: `select_item()`
- Notes: Required by SDL widget callback interface

### w_env_select::set_path
- Signature: `void set_path(const char *p)`
- Purpose: Update the selected file path and reflect it in the button label
- Inputs: File path string
- Outputs/Return: None
- Side effects: Updates `item`, `item_name`, and button display
- Calls: `FileSpecifier::GetName()`, `FileSpecifier::HideExtension()`
- Notes: Strips file extension for display

### w_crosshair_display::w_crosshair_display (constructor)
- Signature: `w_crosshair_display()`
- Purpose: Initialize SDL surface for crosshair preview rendering
- Inputs: None
- Outputs/Return: None (initializes member `surface`)
- Side effects: Allocates 80├ù80 16-bit RGB surface via `SDL_CreateRGBSurface()`
- Calls: `SDL_CreateRGBSurface()`
- Notes: Sets widget dimensions to `kSize` (80); uses 5-5-5 RGB bit layout

### w_crosshair_display::~w_crosshair_display (destructor)
- Signature: `~w_crosshair_display()`
- Purpose: Release SDL surface memory
- Inputs: None
- Outputs/Return: None
- Side effects: Frees `surface` via `SDL_FreeSurface()`
- Calls: `SDL_FreeSurface()`
- Notes: Sets surface pointer to null after freeing

### w_crosshair_display::draw
- Signature: `void draw(SDL_Surface *s) const`
- Purpose: Render the crosshair preview onto the display surface
- Inputs: Target SDL surface `s`
- Outputs/Return: None
- Side effects: Fills preview surface, renders crosshair, blits to target surface
- Calls: `SDL_FillRect()`, `draw_rectangle()`, `Crosshairs_IsActive()`, `Crosshairs_SetActive()`, `Crosshairs_Render()`, `SDL_BlitSurface()`
- Notes: Temporarily activates crosshairs for rendering, then restores previous state; uses theme colors for background and frame

## Control Flow Notes
This file provides widgets that integrate into a preferences/configuration UI dialog system (not the main render loop). The environment selector (`w_env_select::select_item`) runs a modal dialog and blocks until user selection or cancellation. The crosshair display is a passive render-on-demand widget called during preferences UI painting.

## External Dependencies
- **SDL**: Surface allocation/rendering (`SDL_CreateRGBSurface`, `SDL_FreeSurface`, `SDL_FillRect`, `SDL_BlitSurface`, `SDL_Rect`)
- **Crosshairs module** (`Crosshairs.h`): `Crosshairs_IsActive()`, `Crosshairs_SetActive()`, `Crosshairs_Render()`
- **File system utilities**: `FileSpecifier`, `DirectorySpecifier`, `FindAllFiles`, `FindThemes`
- **UI framework**: `dialog`, `widget`, `w_select_button`, `w_list<T>`, `w_title`, `w_spacer`, `w_button`, `vertical_placer`
- **Theming**: `get_theme_color()` (defined elsewhere)
- **Screen/drawing**: `clear_screen()`, `draw_rectangle()`, `draw_text()`, `set_drawing_clip_rectangle()` (defined elsewhere)
