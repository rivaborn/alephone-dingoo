# Source_Files/Misc/sdl_widgets.h

## File Purpose
Comprehensive widget framework for SDL-based dialog UI. Defines base widget class, specific widget types (buttons, text entry, lists, toggles, sliders), and wrapper/adapter classes for MVC-style data binding. Supports themed rendering, callbacks, and integration with dialog system.

## Core Responsibilities
- Define abstract widget base class and interface for all UI controls
- Implement specific widget types: buttons, text entry, selection lists, toggles, sliders, color pickers, file choosers
- Support event handling (mouse, keyboard, custom events) and input dispatch
- Provide callback mechanisms for user actions and state changes
- Enable widget identification, enabling/disabling, and dynamic label management
- Support layout and positioning integration through `placeable` interface
- Provide template-based list widgets for arbitrary item types
- Implement specialized widgets for metaserver networking (game lists, player lists)
- Offer wrapper classes for data binding and MVC patterns

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `widget` | class | Abstract base for all widgets; defines interface for drawing, input handling, enabling, identification |
| `w_button_base` | class | Base for button variants; handles rendering, mouse interaction, callbacks |
| `w_select` | class | Dropdown selection with dynamic labels and selection-changed callbacks |
| `w_text_entry` | class | Text input field with enter/value-changed callbacks |
| `w_toggle` | class | On/off toggle; subclass of w_select with predefined labels |
| `w_enabling_toggle` | class | Toggle that controls enabled state of dependent widgets |
| `w_list_base` | class | Base for scrollable list widgets; handles scrolling, selection, mouse interaction |
| `w_list<T>` | class template | Typed list for arbitrary item types |
| `w_slider` | class | Horizontal slider for numeric selection |
| `w_items_in_room<T>` | class template | Generic list for metaserver items (games, players); draws items with callback on selection |
| `w_games_in_room` | class | Specialized list displaying GameListMessage entries |
| `w_players_in_room` | class | Specialized list displaying MetaserverPlayerInfo entries |
| `ColoredChatEntry` | struct | Chat message with type, sender color/name, message text |
| `w_colorful_chat` | class | Colored chat display list |
| `ToggleWidget` | class | Adapter wrapping w_toggle for data binding |
| `SelectorWidget` | class | Abstract adapter for selection widgets with data binding |
| `ButtonWidget` | class | Adapter wrapping w_button_base for callbacks |
| `EditTextWidget` | class | Adapter wrapping w_text_entry for text data binding |
| `FileChooserWidget` | class | Adapter wrapping w_file_chooser for FileSpecifier binding |
| `GameListWidget` | class | Adapter wrapping w_games_in_room with callback dispatch |
| `ControlHitCallback` | typedef | `boost::function<void (void)>` for control activation |
| `selection_changed_callback_t` | typedef | `boost::function<void (w_select*)>` for selection changes |

## Global / File-Static State
None (all state is instance-based; globals defined in sdl_dialogs.h: `top_dialog`).

## Key Functions / Methods

### widget::draw
- **Signature**: `virtual void draw(SDL_Surface *s) const = 0`
- **Purpose**: Render widget to SDL surface; pure virtual, overridden by subclasses
- **Inputs**: `s` - destination SDL surface
- **Outputs/Return**: None
- **Side effects**: Modifies SDL surface pixel data
- **Calls**: Theme functions (get_theme_font, get_theme_color, get_theme_image) defined elsewhere
- **Notes**: Each widget is responsible for its own appearance; theme system supplies colors, fonts, images

### widget::set_enabled
- **Signature**: `void set_enabled(bool inEnabled)`
- **Purpose**: Enable/disable user interaction; updates enabled flag and associated label
- **Inputs**: `inEnabled` - true to enable, false to disable
- **Outputs/Return**: None
- **Side effects**: Sets `enabled` flag; calls `set_enabled()` on associated label if present
- **Calls**: (none visible in header)
- **Notes**: Widgets should render differently when disabled; `is_selectable()` respects enabled state

### widget::click / event / mouse_*
- **Signature**: `virtual void click(int /*x*/, int /*y*/)`, `virtual void mouse_move(int x, int y)`, `virtual void event(SDL_Event &e)`
- **Purpose**: Handle user input; default implementations do nothing or call click()
- **Inputs**: x, y coordinates or SDL_Event
- **Outputs/Return**: None
- **Side effects**: May trigger callbacks, modify state, request redraw
- **Calls**: Virtual; overridden by specific widget types
- **Notes**: click() is invoked on mouse_down by default; subclasses override for specific behavior

### w_select::get_selection / set_selection
- **Signature**: `size_t get_selection(void) const`, `void set_selection(size_t selection, bool simulate_user_input = false)`
- **Purpose**: Query or change current selection; optionally trigger callbacks
- **Inputs**: selection index, flag to simulate user action
- **Outputs/Return**: Current selection index (UNONE if unknown), or void
- **Side effects**: Calls `selection_changed()` if `simulate_user_input` is true; invokes selection_changed_callback
- **Calls**: `selection_changed()` (virtual), `play_dialog_sound()` (defined elsewhere)
- **Notes**: UNONE (0xFFFFFFFF) means no valid selection; setting with `simulate_user_input=true` triggers callbacks as if user selected

### w_text_entry::set_text / get_text
- **Signature**: `void set_text(const char *text)`, `const char *get_text(void)`
- **Purpose**: Set or retrieve text entry content
- **Inputs**: text pointer (C string)
- **Outputs/Return**: Pointer to internal buffer
- **Side effects**: `set_text()` calls `modified_text()` which triggers callbacks
- **Calls**: `modified_text()` (for set_text)
- **Notes**: Buffer size limited by `max_chars`; get_text returns direct pointer to internal buffer

### w_list_base::get_selection / set_selection
- **Signature**: `size_t get_selection(void)`, `void set_selection(size_t s)`
- **Purpose**: Query or change selected item; handles scrolling to keep selection visible
- **Inputs**: item index
- **Outputs/Return**: Current selection, or void
- **Side effects**: Updates scroll position via `center_item()` or `set_top_item()`; requests redraw
- **Calls**: `new_items()`, `center_item()`, `set_top_item()`
- **Notes**: Scrollbar thumb position automatically calculated based on selection and visible items

### w_file_chooser::set_file / get_file
- **Signature**: `void set_file(const FileSpecifier& inFile)`, `const FileSpecifier& get_file()`
- **Purpose**: Set or retrieve the chosen file
- **Inputs**: FileSpecifier object
- **Outputs/Return**: Reference to internal FileSpecifier
- **Side effects**: `set_file()` calls `update_filename()` which updates displayed text and triggers callback
- **Calls**: `update_filename()`
- **Notes**: File display string extracted from FileSpecifier path

### w_items_in_room<T>::set_collection
- **Signature**: `void set_collection(const std::vector<tElement>& elements)`
- **Purpose**: Replace list contents; used for dynamic updates (e.g., game list refresh)
- **Inputs**: Vector of items
- **Outputs/Return**: None
- **Side effects**: Replaces `m_items`; resets selection; forces dialog redraw via `draw_dirty_widgets()`
- **Calls**: `new_items()`, `get_owning_dialog()->draw_dirty_widgets()`
- **Notes**: Used by GameListWidget and PlayerListWidget to update display from network messages

### w_enabling_toggle::add_dependent_widget
- **Signature**: `void add_dependent_widget(widget* inWidget)`
- **Purpose**: Register a widget to be enabled/disabled by this toggle
- **Inputs**: Pointer to dependent widget
- **Outputs/Return**: None
- **Side effects**: Adds widget to `dependents` set; immediately sets enabled state based on toggle state
- **Calls**: `update_widget_enabled()`
- **Notes**: When toggle changes, all dependents updated automatically via `selection_changed()`

## Control Flow Notes

**Initialization/Setup**:
1. Widgets constructed individually
2. Added to dialog via `dialog::add()`
3. Dialog calls `set_owning_dialog()` on each widget (friend access)
4. Dialog layouts widgets via `place()` on placeable interface
5. Callbacks registered via `set_*_callback()` methods

**Event/Render Loop**:
1. Dialog calls `process_events()` which dispatches SDL events to widgets
2. Widget `event()` or `click()` methods handle input
3. Input may trigger callbacks (action_proc for buttons, selection_changed_callback for selectors)
4. Dialog calls `draw()` on each widget (or `draw_dirty_widgets()` for optimized redraw)
5. Widgets use theme system to get fonts, colors, images before rendering

**Data Binding** (via wrapper classes):
- Wrapper classes (ToggleWidget, EditTextWidget, etc.) inherit from `Bindable<T>` to enable MVC patterns
- Wrappers register callbacks on underlying widgets to sync data on user action
- `bind_export()` / `bind_import()` methods couple widget state to model objects

## External Dependencies

**Framework Integration**:
- `sdl_dialogs.h` ΓÇö provides `dialog` class (friend), `placeable` base class, theme system (get_theme_font, get_theme_color, get_theme_image, get_theme_space)
- `sdl_fonts.h` ΓÇö `font_info` for text rendering
- `screen_drawing.h` ΓÇö `draw_text()`, `draw_polygon()`, `draw_line()`, `draw_rectangle()` utilities

**Core/Data Types**:
- `cseries.h` ΓÇö base types (uint16, int16, etc.), macros (MIN, TEST_FLAG)
- `map.h` ΓÇö `entry_point` struct for w_levels
- `tags.h` ΓÇö `Typecode` enum for w_file_chooser
- `FileHandler.h` ΓÇö `FileSpecifier` class for file operations

**Networking/Metaserver** (conditional: `!DISABLE_NETWORKING`):
- `metaserver_messages.h` ΓÇö `GameListMessage`, `MetaserverPlayerInfo`, `GameListMessage::GameListEntry`
- `network.h` ΓÇö `prospective_joiner_info` for w_joining_players_in_room

**Boost**:
- `boost/function.hpp` ΓÇö callback function objects
- `boost/bind.hpp` ΓÇö method binding for callbacks

**SDL**:
- `<SDL.h>` ΓÇö surface, events, rendering primitives
- `<vector>`, `<set>` ΓÇö STL containers for items and dependents

**Defined Elsewhere** (not in this file):
- `dialog::activate_widget()`, `dialog::draw_dirty_widgets()`, `dialog::get_owning_dialog()`
- `play_dialog_sound()`, `get_dialog_player_color()` ΓÇö sound and color utilities
- `load_sdl_font()` ΓÇö font loading
- `set_drawing_clip_rectangle()`, `draw_text()` ΓÇö rendering primitives
- `get_theme_space()` with theme widget enum values (SPACER_WIDGET, MESSAGE_WIDGET, etc.)
