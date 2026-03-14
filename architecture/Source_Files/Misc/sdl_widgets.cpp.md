# Source_Files/Misc/sdl_widgets.cpp

## File Purpose
SDL-based GUI widget implementation for the Aleph One game engine. Provides a complete widget system (buttons, text entry, lists, tabs, sliders) for constructing dialog boxes. Includes specialized widgets for game-specific features (level selection, player rosters, network game listings, chat).

## Core Responsibilities
- Base widget class setup: initialization, active/enabled state, theme-based styling, layout integration
- Text rendering: static text, labels, multi-style formatted text, password masking
- Input widgets: buttons, text entry (normal/numeric/password), key binding, sliders, color/player color selection
- Selection widgets: dropdown-like buttons, cycling selectors, popup menus, toggle switches
- Container widgets: tab groups with multi-pane support, scrollable lists with thumb-based scrolling
- Specialized list widgets: game levels, string lists, network games, player rosters, colored chat
- Network UI widgets: game browser, player list, joining player tracker with postgame carnage report visualization
- Wrapper/adapter layer: SDL-widget bridge classes for common patterns (toggles, selectors, buttons, text fields)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `widget` | class | Base class for all UI widgets; manages enabled state, focus, dirty flag, bounds, theme font/style |
| `w_static_text` | class | Non-interactive text display; owns string copy; supports dynamic text updates |
| `w_label` | class | Label that can delegate clicks to associated widget; styled per theme |
| `w_button_base` | class | Button foundation with theme images, mouse tracking, pressed/active state, callback system |
| `w_tab` | class | Multi-tab interface; manages tab layout, selection state, keyboard navigation |
| `w_select` | class | Cycling selection widget; advances through label array on click or keyboard (LEFT/RIGHT) |
| `w_toggle` | class | On/off binary select; extends `w_select` with preset on/off labels |
| `w_enabling_toggle` | class | Toggle that enable/disable dependent widgets based on state |
| `w_text_entry` | class | Text input field; supports undo/redo, Mac Roman, UTF-8; callbacks on enter/value change |
| `w_key` | class | Key/mouse-button binding widget; enters binding mode on click, captures next key press |
| `w_slider` | class | Horizontal slider; thumb dragging, discrete item selection |
| `w_list_base` | class | Template-free list foundation; handles scrolling, thumb, item selection, wheel events |
| `w_list<T>` | template class | Generic list for vector<T>; abstract draw_item() override per item type |
| `w_levels` | class | `w_list<entry_point>`; displays game levels with optional level numbers |
| `w_games_in_room` | class | Network game browser; 3-line item layout with running/incompatible state coloring |
| `w_players_in_room` | class | Player list with color/team swatches and away status dimming |
| `w_colorful_chat` | class | `w_list<ColoredChatEntry>`; colored sender swatches, message styling, team/server/local indicators |
| `ColoredChatEntry` | struct | Chat message with type (ChatMessage/PrivateMessage/ServerMessage/LocalMessage), sender color, sender name, message text |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sNoValidOptionsString` | `const char*` | static | Fallback text when `w_select` has no labels |
| `sFileChooserInvalidFileString` | `const char*` | static | Fallback text when no file selected in `w_file_chooser` |

## Key Functions / Methods

### widget::widget() / widget::widget(int theme_widget)
- **Signature:** `widget::widget()` and `widget::widget(int theme_widget)`
- **Purpose:** Initialize widget with default/theme-based font, enable state, and layout placeholders
- **Inputs:** Optional theme widget type ID
- **Outputs/Return:** Constructed `widget` object; all fields zeroed except those initialized in member list
- **Side effects (global state, I/O, alloc):** Calls `get_theme_font(theme_widget, style)` to load font if theme provided
- **Calls:** `get_theme_font()`
- **Notes:** Base class; `font`, `style`, `identifier`, `owning_dialog` set to NULL/0; `active`, `dirty`, `enabled` initialized; `rect` zeroed

### widget::set_enabled(bool inEnabled)
- **Signature:** `void widget::set_enabled(bool inEnabled)`
- **Purpose:** Enable/disable user interaction; propagate to associated label
- **Inputs:** `inEnabled` ΓÇô true to enable, false to disable
- **Outputs/Return:** None
- **Side effects:** Sets `enabled` flag; if disabled and widget had focus, calls `owning_dialog->activate_next_widget()`; marks `dirty`; calls `associated_label->set_enabled()` if present
- **Calls:** `owning_dialog->activate_next_widget()`, `w_label::set_enabled()`
- **Notes:** Removes focus on disable to avoid stuck focus on disabled widget

### widget::place(const SDL_Rect &r, placement_flags flags)
- **Signature:** `void widget::place(const SDL_Rect &r, placement_flags flags)`
- **Purpose:** Position and size widget in parent layout; respect saved min width and alignment flags
- **Inputs:** `r` ΓÇô target rect; `flags` ΓÇô alignment (kFill, kAlignLeft, kAlignCenter, kAlignRight)
- **Outputs/Return:** Modifies `rect` member
- **Side effects:** Sets widget bounds based on flags
- **Calls:** None
- **Notes:** If `kFill`, uses full width; else centers, left-aligns, or right-aligns based on flags and `saved_min_width`

### w_button_base::draw(SDL_Surface *s) const
- **Signature:** `void w_button_base::draw(SDL_Surface *s) const`
- **Purpose:** Render button with state-specific theme images or solid color; draw text centered
- **Inputs:** `s` ΓÇô target SDL surface
- **Outputs/Return:** Blits button parts (left, center, right) and text to surface
- **Side effects (global state, I/O, alloc):** Reads theme colors/images; calls SDL_BlitSurface, draw_text
- **Calls:** `use_theme_images()`, `get_theme_image()`, `get_theme_color()`, `SDL_BlitSurface()`, `draw_text()`
- **Notes:** State determined by `pressed`, `enabled`, `active`; button is 3-part (left cap, center tile/stretch, right cap); text is offset by button spacing constants

### w_button_base::mouse_up(int x, int y)
- **Signature:** `void w_button_base::mouse_up(int x, int y)`
- **Purpose:** Complete mouse button release; fire callback if released inside button bounds
- **Inputs:** `x`, `y` ΓÇô mouse coordinates relative to button
- **Outputs/Return:** None
- **Side effects:** Clears `down` and `pressed`; marks `dirty`; calls `proc(arg)` callback if x/y in bounds
- **Calls:** `get_owning_dialog()->draw_dirty_widgets()`, callback `proc(arg)`
- **Notes:** Callback only fires if release is inside button; coordinate system is local to button rect

### w_select::click(int x, int y)
- **Signature:** `void w_select::click(int x, int y)`
- **Purpose:** Advance selection to next option on click
- **Inputs:** `x`, `y` ΓÇô ignored
- **Outputs/Return:** None
- **Side effects:** Increments `selection`; wraps to 0 if past end; calls `selection_changed()`
- **Calls:** `selection_changed()`
- **Notes:** Allows cycling through a fixed list of options

### w_select::event(SDL_Event &e)
- **Signature:** `void w_select::event(SDL_Event &e)`
- **Purpose:** Handle LEFT/RIGHT arrow keys to navigate selection
- **Inputs:** `e` ΓÇô SDL keyboard event
- **Outputs/Return:** None
- **Side effects:** Updates `selection` on LEFT/RIGHT; consumes event by setting `e.type = SDL_NOEVENT`
- **Calls:** `selection_changed()`
- **Notes:** Wraps at boundaries; LEFT goes backward, RIGHT goes forward

### w_list_base::set_selection(size_t s)
- **Signature:** `void w_list_base::set_selection(size_t s)`
- **Purpose:** Select an item and ensure it is visible (scrolled into view)
- **Inputs:** `s` ΓÇô item index (assert 0 Γëñ s < num_items)
- **Outputs/Return:** None
- **Side effects:** Updates `selection`; calls `set_top_item()` to scroll if selection is off-screen; marks `dirty`
- **Calls:** `set_top_item()`
- **Notes:** Automatically scrolls list to make selection visible

### w_list_base::set_top_item(size_t i)
- **Signature:** `void w_list_base::set_top_item(size_t i)`
- **Purpose:** Scroll list to show item at top; recalculate scroll thumb position
- **Inputs:** `i` ΓÇô first visible item index
- **Outputs/Return:** None
- **Side effects:** Updates `top_item`; recomputes `thumb_y` position based on scroll ratio; marks `dirty`
- **Calls:** `PIN()` (clipping macro)
- **Notes:** Clamped to valid range; thumb position proportional to list scroll

### w_games_in_room::draw_item(const GameListMessage::GameListEntry& item, SDL_Surface* s, int16 x, int16 y, uint16 width, bool selected) const
- **Signature:** As above
- **Purpose:** Render a network game entry (3 lines: name+time-remaining, game+map, player-count+settings)
- **Inputs:** `item` ΓÇô game info; `s` ΓÇô surface; `x`, `y`, `width` ΓÇô bounds; `selected` ΓÇô highlight flag
- **Outputs/Return:** Draws to surface
- **Side effects:** Fills background, draws color-coded text (based on game state: running, compatible, incompatible), clips text to bounds
- **Calls:** `SDL_FillRect()`, `get_theme_color()`, `use_theme_color()`, `draw_rectangle()`, `font->draw_styled_text()`, `set_drawing_clip_rectangle()`, `text_width()`
- **Notes:** State-dependent coloring (SELECTED_GAME, RUNNING_GAME, INCOMPATIBLE_GAME, etc.); time remaining or ping shown on first line

### w_players_in_room::draw_item(const MetaserverPlayerInfo& item, SDL_Surface* s, int16 x, int16 y, uint16 width, bool selected) const
- **Signature:** As above
- **Purpose:** Render player entry with player-color swatch, team-color swatch, and name
- **Inputs:** Player info, surface, bounds, selected flag
- **Outputs/Return:** Draws to surface
- **Side effects:** Fills color swatches (player color, team color); lightens colors if player is target, darkens if away
- **Calls:** `SDL_MapRGB()`, `SDL_FillRect()`, `lighten()`, `darken()`, `font->draw_styled_text()`
- **Notes:** Away players rendered in gray; selected player highlighted; taper effect between swatches

### w_colorful_chat::append_entry(const ColoredChatEntry& e)
- **Signature:** `void w_colorful_chat::append_entry(const ColoredChatEntry& e)`
- **Purpose:** Add a chat message; wrap long messages to fit available width; auto-scroll to bottom
- **Inputs:** `e` ΓÇô chat entry (message, sender, color, type)
- **Outputs/Return:** None
- **Side effects:** Appends entry to `entries` vector; recalculates `num_items`, scrollbars; preserves scroll position if user was not at bottom
- **Calls:** `font->trunc_styled_text()`, `new_items()`, `set_top_item()`, `append_entry()` (recursive for wrapped lines)
- **Notes:** Recursive to handle multi-line wrapping; preserves user scroll position unless scrolled to bottom

## Control Flow Notes
- **Initialization:** Dialog loads theme assets (images, colors, fonts) once; widgets reference shared theme data
- **Layout phase:** Dialog calls `place()` on each widget in order (row/column/table layout patterns)
- **Render phase:** Dialog iterates widgets and calls `draw()` for each that is visible and dirty; updates clipped SDL rects
- **Event phase:** Dialog dispatches mouse/keyboard events to active widget first; widgets can consume events (set `SDL_NOEVENT`) or let them bubble
- **Callback phase:** Widgets trigger callbacks (e.g., button click, selection change, text entry) after processing input
- **Dirty tracking:** Widgets set `dirty` flag to trigger redraw; dialog's `draw_dirty_widgets()` redraws only marked widgets for efficiency

## External Dependencies
- **SDL.h, SDL_surface.h, SDL_events.h** ΓÇô graphics/input primitives (surfaces, rects, events)
- **sdl_dialogs.h** ΓÇô dialog framework (`dialog`, `placeable`, layout classes)
- **sdl_fonts.h** ΓÇô font rendering (`font_info`, text measurement)
- **screen_drawing.h** ΓÇô drawing utilities (`draw_text`, `draw_rectangle`, `set_drawing_clip_rectangle`)
- **resource_manager.h** ΓÇô load resources (PICT images)
- **network_dialog_widgets_sdl.h** ΓÇô network-specific widgets (used for metaserver lists)
- **metaserver_messages.h, network.h** ΓÇô game/player data structures (GameListMessage, prospective_joiner_info, MetaserverPlayerInfo)
- **Tags/FileHandler** ΓÇô file spec / type code support (w_file_chooser)
- **SoundManager.h** ΓÇô (included but not directly used in visible code)
- **Boost** ΓÇô function and bind for callbacks (`boost::function`, `boost::bind`)
