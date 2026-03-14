# Source_Files/Misc/sdl_dialogs.h

## File Purpose
Header file defining the SDL dialog and widget layout system for the Aleph One engine. Implements a hierarchical layout engine with placeable widgets, modal dialog handling, theme support, and event processing infrastructure.

## Core Responsibilities
- Define the **dialog** class for modal UI presentation and event dispatch
- Define the **placeable** hierarchy (base class + placer subclasses) for widget layout and positioning
- Manage dialog state (active widget, result codes, visibility)
- Handle dialog frame rendering and dirty-widget redraw optimization
- Define theme and widget rendering constants (widget types, states, colors, spaces)
- Provide callbacks and sound definitions for user interaction

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| placeable | class | Abstract base for anything that can be placed in a layout (widgets, placers); tracks visibility |
| widget_placer | class | Abstract placer that owns child placeables and manages their layout |
| tab_placer | class | Placer that manages mutually-exclusive tabs; only active tab is visible |
| table_placer | class | Grid-based placer with configurable columns, spacing, width balancing |
| vertical_placer | class | Stack placeables vertically with configurable spacing and min width |
| horizontal_placer | class | Stack placeables horizontally with configurable spacing, width balancing |
| dialog | class | Top-level modal dialog; runs event loop, manages active/mouse widgets, owns frame surfaces |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| top_dialog | dialog* | global (extern) | Points to active top-level dialog, NULL if none |
| dialog::sKeyRepeatActive | bool | static | SDL key-repeat state (SDL doesn't store this) |
| DIALOG_*_SOUND | #define | global | Maps dialog events to sound IDs (_snd_*) |

## Key Functions / Methods

### dialog::run
- **Signature**: `int run(bool intro_exit_sounds = true)`
- **Purpose**: Execute dialog modally; block until dialog is finished
- **Inputs**: Whether to play intro/exit sounds
- **Outputs/Return**: Dialog result code (set via `quit()`)
- **Side effects**: Saves/restores cursor visibility, keyboard state; draws dialog; processes events until `done` flag set
- **Calls**: `start()`, `process_events()`, `finish()`
- **Notes**: Primary entry point for modal interaction

### dialog::process_events
- **Signature**: `bool process_events()`
- **Purpose**: Process one frame of pending SDL events
- **Inputs**: None (consumes SDL event queue)
- **Outputs/Return**: Boolean "done" flag (true if dialog should close)
- **Side effects**: Updates active/mouse widgets, calls widget event handlers, invokes custom processing callback
- **Calls**: Widget event methods, custom `processing_function` callback

### dialog::draw / dialog::draw_dirty_widgets
- **Signature**: `void draw(void)` / `void draw_dirty_widgets() const`
- **Purpose**: Redraw entire dialog (all widgets) or only widgets marked dirty
- **Side effects**: Updates SDL surface via `update()`
- **Calls**: `draw_widget()` for each widget
- **Notes**: `draw_dirty_widgets()` is called automatically on events; external code can call it for timer-based/network-driven updates

### dialog::activate_widget
- **Signature**: `void activate_widget(widget *w, bool draw = false)` / `void activate_widget(size_t num, bool draw = true)`
- **Purpose**: Transfer focus to specified widget, deactivate previous
- **Side effects**: Sets `active_widget` and `active_widget_num`; optionally redraws

### placeable::place
- **Signature**: `virtual void place(const SDL_Rect &r, placement_flags flags = kDefault) = 0`
- **Purpose**: Position this placeable within rect `r` according to alignment flags
- **Notes**: Pure virtual; overridden by widget_placer subclasses to recursively position children

### tab_placer::choose_tab
- **Signature**: `void choose_tab(int new_tab)`
- **Purpose**: Switch active tab; hides old, shows new
- **Side effects**: Updates visibility of child placeables

## Control Flow Notes
Dialogs are run modally via `run()`, which calls `start()` (setup), then loops `process_events()` until `done` flag is set, then `finish()` (cleanup). The layout system is invoked during `start()` via `new_layout()`, which recursively positions all widgets using the placeable hierarchy. Event dispatch flows: SDL event ΓåÆ `event()` method ΓåÆ widget-specific handlers (click, mouse_move, etc.). Dirty-widget redrawing is triggered by events or can be manually invoked for real-time updates.

## External Dependencies
- **Includes**: `<vector>`, `<memory>`, `boost/function.hpp`; SDL (SDL_Surface, SDL_Rect, SDL_Event forward-declared)
- **Types used, defined elsewhere**: `widget`, `font_info`, `FileSpecifier`, `XML_ElementParser`
- **Extern symbols**: `initialize_dialogs()`, `load_theme()`, `get_theme_font()`, `get_theme_color()`, `get_theme_image()`, `play_dialog_sound()`, `get_dialog_player_color()`, `Theme_GetParser()`
