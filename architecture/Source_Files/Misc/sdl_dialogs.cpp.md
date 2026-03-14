# Source_Files/Misc/sdl_dialogs.cpp

## File Purpose
SDL-based dialog and widget management system for the Aleph One game engine. Handles modal dialogs, event processing, theming (colors/images/fonts), and widget layout/rendering. Theme data is loaded from XML and applied to standard widget types.

## Core Responsibilities
- Dialog lifecycle management (init, display, event loop, cleanup)
- Widget management and hierarchical layout/placement
- Theme loading from MML/XML with per-widget-state styling
- Dialog rendering to offscreen surface; blitting to screen/OpenGL
- Input event dispatching (keyboard, mouse) to active widgets
- Widget activation/focus management
- Dynamic dirty-region redrawing optimization

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `dialog` | class | Container for widgets; manages layout, rendering, event dispatch, modal loop |
| `dialog_image_spec_type` | struct | Theme image metadata: name + scale-to-fit flag |
| `theme_state` | struct | Per-widget-state styling: colors map, image specs map, loaded images map |
| `theme_widget` | struct | Complete theme for one widget type: states, font, spacing |
| `widget_placer` / `tab_placer` / `table_placer` / `vertical_placer` / `horizontal_placer` | classes | Layout containers; calculate min dimensions and position child placeables |
| `XML_ImageParser`, `XML_DColorParser`, `XML_DFontParser`, `XML_DFrameParser` | classes | XML parsing for theme elements (images, colors, fonts, frame spacing) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `top_dialog` | `dialog*` | global | Active dialog (NULL if none); used to prevent re-entrancy |
| `dialog_surface` | `SDL_Surface*` | static | Offscreen buffer (640├ù480 RGB555); dialogs render here, then blit to screen |
| `default_image` | `SDL_Surface*` | static | 1├ù1 magenta fallback image for missing theme resources |
| `theme_resources` | `OpenedResourceFile` | static | Loaded resource file for theme (images, fonts) |
| `dialog_theme` | `std::map<int, theme_widget>` | static | Theme data indexed by widget type; populated by XML parsers |
| `dialog::sKeyRepeatActive` | bool (static) | class member | SDL key repeat state (not stored by SDL itself) |

## Key Functions / Methods

### initialize_dialogs
- Signature: `void initialize_dialogs(FileSpecifier &theme)`
- Purpose: Set up dialog system at startup; allocate surfaces and load theme
- Inputs: `theme` = FileSpecifier for theme file
- Outputs/Return: (void)
- Side effects: Allocates `dialog_surface`, `default_image`; loads theme via `load_theme()`; registers `shutdown_dialogs` via `atexit()`
- Calls: `SDL_CreateRGBSurface()`, `SDL_SetColorKey()`, `load_theme()`, `get_default_theme_spec()`, `write_preferences()`
- Notes: Creates RGB555 surface for compatibility with OpenGL blitting; retries with default theme if load fails

### shutdown_dialogs
- Signature: `static void shutdown_dialogs(void)`
- Purpose: Clean up theme and surface resources on exit
- Inputs: (none)
- Outputs/Return: (void)
- Side effects: Calls `unload_theme()`
- Calls: `unload_theme()`
- Notes: Invoked via atexit hook; not called directly

### dialog::draw
- Signature: `void dialog::draw(void)`
- Purpose: Render entire dialog to offscreen surface and blit to screen/OpenGL
- Inputs: (none; uses member `rect`, `widgets`, `layout_for_fullscreen`)
- Outputs/Return: (void)
- Side effects: Fills `dialog_surface`; blits to video surface or via OpenGL; updates screen
- Calls: `layout()`, `SDL_FillRect()`, `draw_frame_image()`, `draw_widget()`, `update()`, `OGL_Blitter::*`
- Notes: Checks fullscreen state and re-lays out if changed; draws frame from images or rectangle depending on theme

### dialog::event
- Signature: `void dialog::event(SDL_Event &e)`
- Purpose: Dispatch input events to widgets and handle dialog-level hotkeys
- Inputs: `e` = SDL event
- Outputs/Return: (void)
- Side effects: Modifies `active_widget`, `mouse_widget`; calls widget event handlers; may call `quit()`, `draw()`, `toggle_fullscreen()`
- Calls: `active_widget->event()`, `find_widget()`, `activate_*_widget()`, `draw_dirty_widgets()`, `draw()`
- Notes: Handles Alt+Return (fullscreen), Escape (quit), Tab/arrows (focus), Enter (click active); first-pass events to active widget before dispatch

### dialog::run
- Signature: `int dialog::run(bool intro_exit_sounds = true)`
- Purpose: Run modal dialog loop; return when user quits
- Inputs: `intro_exit_sounds` = whether to play entry/exit sounds
- Outputs/Return: `result` code (0 = OK, -1 = Cancel, or custom value from `quit(r)`)
- Side effects: Calls `start()`, enters event loop, calls `finish()`, sets/restores UI state
- Calls: `start()`, `process_events()`, `processing_function` callback, `global_idle_proc()`, `finish()`
- Notes: Modal; blocks caller until dialog closed; sleeps 10ms per frame; respects custom processing function

### dialog::activate_widget
- Signature: `void dialog::activate_widget(size_t num, bool draw = true)` and overload `void dialog::activate_widget(widget *w, bool draw = false)`
- Purpose: Give input focus to a selectable widget; deactivate previous
- Inputs: Widget index or pointer; `draw` = redraw after activation
- Outputs/Return: (void)
- Side effects: Sets `active_widget`, `active_widget_num`; marks widget and associated label as `active = true`; may draw
- Calls: `deactivate_currently_active_widget()`, `draw_widget()`
- Notes: Skips non-selectable or invisible widgets; special handling for labels associated with widgets (activates associated widget instead)

### dialog::draw_dirty_widgets
- Signature: `void dialog::draw_dirty_widgets() const`
- Purpose: Redraw only widgets marked dirty (optimization for incremental updates)
- Inputs: (none; uses member `widgets`)
- Outputs/Return: (void)
- Side effects: Calls `draw_widget()` for each dirty, visible widget
- Calls: `draw_widget()`
- Notes: Only runs if this dialog is `top_dialog`; used for responsive UI without full redraws

### dialog::layout
- Signature: `void dialog::layout(void)`
- Purpose: Calculate dialog size and position widgets using placer
- Inputs: (none; uses `placer`)
- Outputs/Return: (void)
- Side effects: Sets `rect.w`, `rect.h`, `rect.x`, `rect.y`; calls `placer->place()`; caches fullscreen state
- Calls: `placer->min_width()`, `placer->min_height()`, `placer->place()`, `SDL_GetVideoSurface()`, `get_theme_space()`, `get_screen_mode()`
- Notes: Centers dialog on video surface; includes theme frame spacing; re-runs if fullscreen toggle detected

---

## Control Flow Notes
- **Init**: `initialize_dialogs()` ΓåÆ allocates surfaces, loads theme XML, registers cleanup
- **Modal loop**: `dialog::run()` ΓåÆ `start()` [layout, draw] ΓåÆ loop `process_events()` ΓåÆ `event()` [dispatch] ΓåÆ optional `processing_function()` ΓåÆ `finish()` [cleanup, restore parent dialog]
- **Rendering**: `dialog::draw()` or `draw_dirty_widgets()` ΓåÆ render to `dialog_surface` ΓåÆ blit via SDL or OpenGL `OGL_Blitter`
- **Widget lifecycle**: `add()` ΓåÆ managed in `widgets` vector ΓåÆ activated via keyboard/mouse ΓåÆ `event()` calls ΓåÆ widgets mark themselves dirty ΓåÆ redrawn next frame
- **Theme**: XML parsers populate static `dialog_theme` map at load time; accessed via `get_theme_color()`, `get_theme_image()`, `get_theme_font()`, `get_theme_space()`

## External Dependencies
- **Includes**: SDL (video, events, surfaces), SDL_opengl, font system (sdl_fonts.h), widget system (sdl_widgets.h), shape/screen drawing, XML parsing (XML_Loader_SDL, XML_ParseTreeRoot), preferences, SoundManager
- **Defined elsewhere**: `widget`, `placeable`, `font_info`, `FileSpecifier`, `OGL_Blitter`, `OGL_Setup`, `get_default_theme_spec()`, `load_theme()`, `play_dialog_sound()`, `global_idle_proc()`, `toggle_fullscreen()`, `clear_screen()`, `dump_screen()`, `SDL_ShowCursor()`, `SDL_EnableUNICODE()`, `SDL_EnableKeyRepeat()`
- **Notable**: Dialog rendering bridges SDL and OpenGL; surfaces are RGB555 to match OpenGL output
