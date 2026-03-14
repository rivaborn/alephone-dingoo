# Source_Files/Misc/preferences_widgets_sdl.h

## File Purpose
Defines SDL-specific UI widgets for game preferences, particularly for theme and environment file selection in the Aleph One engine. Provides file-browsing widgets with callback support for notifying parent dialogs when selections are made.

## Core Responsibilities
- Theme discovery: enumerate and collect theme files from the filesystem
- Environment item representation: struct wrapping file paths with UI metadata (indentation, selectability)
- File list widget: display environment/theme items with hierarchy indentation and selective enablement
- File selection widget: button that opens a file selection dialog and displays the chosen file
- Crosshair display: render a crosshair graphic in a widget-compatible format

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `FindThemes` | class | `FileFinder` subclass; locates `theme2.mml` files in directories and collects their parent paths into a vector |
| `env_item` | struct | Represents a single file or directory entry with UI state: path, display name, indentation level, selectability flag |
| `w_env_list` | class | Template-instantiated list widget; displays `env_item` entries with theme-colored rendering and directory/file distinction |
| `w_env_select` | class | File selection button widget; manages a `FileSpecifier`, triggers file selection dialogs, supports callback on user selection |
| `w_crosshair_display` | class | Non-interactive widget; renders a fixed 80├ù80 SDL surface displaying a crosshair graphic |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | extern | Search path for game data files; imported from `shell_sdl.cpp` |

## Key Functions / Methods

### FindThemes::found
- **Signature:** `bool found(FileSpecifier &file)` (private)
- **Purpose:** Callback invoked by `FileFinder::Find()` for each discovered file; selects only `theme2.mml` files and collects their parent directory paths
- **Inputs:** `FileSpecifier &file` ΓÇô candidate file found during directory traversal
- **Outputs/Return:** `false` always (permits continued enumeration)
- **Side effects:** Appends parent path to `dest_vector` if filename matches `theme2.mml`
- **Calls:** `FileSpecifier::SplitPath()`, `vector::push_back()`
- **Notes:** Matches `theme2.mml` specifically; ignores `theme.mml` or other files

### w_env_list::w_env_list
- **Signature:** `w_env_list(const vector<env_item> &items, const char *selection, dialog *d)`
- **Purpose:** Initialize list widget; set up initial selection based on file path matching
- **Inputs:** `items` ΓÇô vector of `env_item` entries; `selection` ΓÇô file path to match for initial selection; `d` ΓÇô parent dialog
- **Outputs/Return:** None
- **Side effects:** Sets internal selection index and stores parent dialog pointer
- **Calls:** Constructor chain, `FileSpecifier::GetPath()`, `set_selection()`, `center_item()`

### w_env_list::draw_item
- **Signature:** `void draw_item(vector<env_item>::const_iterator i, SDL_Surface *s, int16 x, int16 y, uint16 width, bool selected) const`
- **Purpose:** Render a single list item; applies theme colors (selectable items use `ITEM_WIDGET` colors; directories use `LABEL_WIDGET`)
- **Inputs:** Iterator to item; SDL surface; x/y position; width; selection state
- **Outputs/Return:** Draws directly to SDL surface
- **Side effects:** Sets drawing clip rectangle; modifies surface pixel data
- **Calls:** `get_theme_color()`, `set_drawing_clip_rectangle()`, `draw_text()`, `FileSpecifier::HideExtension()`
- **Notes:** Indentation achieved by offsetting x by `indent * 8` pixels

### w_env_select::w_env_select
- **Signature:** `w_env_select(const char *path, const char *m, Typecode t, dialog *d)`
- **Purpose:** Initialize file selection button; set initial file path and configure callback mechanism
- **Inputs:** `path` ΓÇô initial file path; `m` ΓÇô dialog menu title; `t` ΓÇô file type code; `d` ΓÇô parent dialog
- **Outputs/Return:** None
- **Side effects:** Calls `set_path()`, registers this object as callback argument
- **Calls:** Parent `w_select_button` constructor, `set_arg()`, `set_path()`
- **Notes:** Stores parent dialog, file type, and callback function pointer (initially `NULL`)

### w_env_select::set_selection_made_callback
- **Signature:** `void set_selection_made_callback(selection_made_callback_t inCallback)`
- **Purpose:** Install a callback function to be invoked when user confirms a file selection
- **Inputs:** Function pointer of type `selection_made_callback_t` (typedef'd as `void (*)(w_env_select*)`)
- **Outputs/Return:** None
- **Side effects:** Stores callback in `mCallback` member
- **Calls:** None
- **Notes:** Callback receives pointer to this widget; allows parent dialog to react to selection

### w_crosshair_display::w_crosshair_display
- **Signature:** `w_crosshair_display()` (constructor)
- **Purpose:** Initialize crosshair display widget; allocate internal SDL surface
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Creates and stores an SDL surface for crosshair rendering
- **Calls:** Defined elsewhere (implementation in `.cpp` file)

### w_crosshair_display::draw
- **Signature:** `void draw(SDL_Surface *s) const`
- **Purpose:** Render the internal crosshair surface to the given display surface
- **Inputs:** Target SDL surface `s`
- **Outputs/Return:** Renders directly to surface
- **Side effects:** Modifies target surface pixels
- **Calls:** Defined elsewhere (implementation in `.cpp` file)

## Control Flow Notes
This file defines UI widgets used in preference/settings dialogs. Initialization occurs when dialogs are constructed; rendering happens during dialog draw cycles. The `w_env_select` widget integrates with the shell's file-browser and game's data-search-path mechanism to locate theme and environment files. Callbacks allow parent dialogs to respond to user selections without coupling widgets directly to dialog logic.

## External Dependencies
- **Includes:** `cseries.h` (base types and macros), `find_files.h` (`FileFinder`, `Typecode`), `collection_definition.h`, `sdl_widgets.h` (base widget classes), `sdl_fonts.h` (font metrics), `screen.h`, `screen_drawing.h` (drawing utilities), `interface.h`
- **Defined elsewhere:** `FileFinder::Find()`, `FileSpecifier::*()` methods, `dialog` class, `get_theme_color()`, `get_theme_font()`, `set_drawing_clip_rectangle()`, `draw_text()`, widget theme constants (`ITEM_WIDGET`, `LABEL_WIDGET`, `ACTIVE_STATE`, `DEFAULT_STATE`)
