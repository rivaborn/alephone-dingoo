# Source_Files/Misc/carbon_widgets.cpp

## File Purpose
Implements Carbon macOS UI widgets for game dialogs, including text fields, selectors, file choosers, and custom-drawn player displays. Bridges game data structures to macOS Carbon Control API.

## Core Responsibilities
- Manage selector (popup menu) labels and values
- Update and retrieve text from edit controls
- Convert between numeric and string representations in number fields
- Integrate file chooser dialogs with game file system
- Render custom player info displays with team/color badges
- Implement color picker dialogs with live preview
- Support show/hide and lifecycle management for composited widgets

## Key Types / Data Structures
None defined in this file (class definitions are in `carbon_widgets.h`).

## Global / File-Static State
None.

## Key Functions / Methods

### SelectorWidget::set_labels
- Signature: `void set_labels(const std::vector<std::string>& labels)`
- Purpose: Populate menu items in a popup control from a label vector
- Inputs: Vector of menu item strings
- Outputs/Return: None (modifies control state)
- Side effects: Clears existing menu items, recreates with new ones via Carbon API
- Calls: `GetControlPopupMenuHandle()`, `CountMenuItems()`, `DeleteMenuItem()`, `CFStringCreateWithCString()`, `AppendMenuItemTextWithCFString()`, `CFRelease()`, `SetControl32BitMaximum()`
- Notes: Manages Carbon resource lifecycle (CFString creation/release)

### EditTextOrNumberWidget::get_text
- Signature: `const string get_text()`
- Purpose: Retrieve current text content from edit control
- Inputs: None
- Outputs/Return: std::string containing text data
- Side effects: Allocates buffer (caller must manage via return)
- Calls: `GetControlDataSize()`, `GetControlData()`
- Notes: Direct memory allocation; size derived from control state

### EditNumberWidget::set_value / get_value
- Signature: `void set_value(int value)` / `int get_value()`
- Purpose: Convert between integer values and text representation in number field
- Inputs: Integer value
- Outputs/Return: Integer from parsed text, or void
- Side effects: Updates control text display
- Calls: `NumToString()`, `StringToNum()`, `pstring_to_string()`, `copy_string_to_pstring()`, `set_text()`
- Notes: Uses Pascal string intermediaries (`ptemporary`); delegates text I/O to parent class

### FileChooserWidget::choose_file
- Signature: `void choose_file()`
- Purpose: Launch file open dialog, update display, trigger callback on success
- Inputs: None (uses m_type and m_prompt member state)
- Outputs/Return: None
- Side effects: Invokes system file dialog; updates m_file and display text; calls user callback
- Calls: `FileSpecifier::ReadDialog()`, `FileSpecifier::GetName()`, `m_text->set_text()`, `m_callback()`
- Notes: Callback execution depends on user confirming dialog

### PlayersInGameWidget::pigDrawer (static)
- Signature: `static void pigDrawer(ControlRef Ctrl, void* ignored)`
- Purpose: Custom Control drawing callback for player list display (2-column grid of player boxes)
- Inputs: Carbon ControlRef, unused context pointer
- Outputs/Return: None (renders directly to port)
- Side effects: Modifies foreground color, draws to current port
- Calls: `GetControlBounds()`, `ForeColor()`, `PaintRect()`, `FrameRect()`, `GetFontInfo()`, `MoveTo()`, `NetNumberOfPlayerIsValid()`, `NetGetNumberOfPlayers()`, `SetRect()`, `OffsetRect()`, `draw_player_box_with_team()`
- Notes: Lays out 2 columns of fixed-size boxes; font metrics determine header height

### draw_player_box_with_team (static helper)
- Signature: `static void draw_player_box_with_team(Rect *rectangle, short player_index)`
- Purpose: Render a single player box with team badge (left) and color badge (right), plus name
- Inputs: Destination rectangle, player index
- Outputs/Return: None
- Side effects: Saves/restores foreground color; draws to current port
- Calls: `NetGetPlayerData()`, `calculate_box_colors()`, `RGBForeColor()`, `PaintRect()`, `MoveTo()`, `LineTo()`, `CopyPascalStringToC()`, `_draw_screen_text()`, `GetForeColor()`
- Notes: Bevel effect drawn via repeated line strokes; team and color determined by player_info struct

### calculate_box_colors (static helper)
- Signature: `static void calculate_box_colors(short color_index, RGBColor *highlight_color, RGBColor *bar_color, RGBColor *shadow_color)`
- Purpose: Derive bar and shadow colors from a highlight color for 3D bevel effect
- Inputs: Color index, pointer to highlight color (output param)
- Outputs/Return: None (writes three RGBColor structures via output pointers)
- Side effects: None
- Calls: `_get_player_color()`
- Notes: Bar = 70% of highlight; Shadow = 20% of highlight; Simple linear scaling

### ListWidgetValueForItem (overloads)
- Signature: Three overloads for `MetaserverPlayerInfo*`, `prospective_joiner_info*`, `GameListMessage::GameListEntry*`
- Purpose: Extract display string from list item (template specialization pattern)
- Inputs: Typed item pointer
- Outputs/Return: std::string
- Side effects: None
- Calls: Element accessors (e.g., `element->name()` or `pstring_to_string()`)
- Notes: Handles NULL inputs gracefully; bridges C++/Pascal string formats

### ColourPickerWidget::chooseColour
- Signature: `void chooseColour()`
- Purpose: Launch macOS color picker dialog, store result in m_colour
- Inputs: None (uses member m_ctrl and m_colour)
- Outputs/Return: None
- Side effects: Updates m_colour on user acceptance
- Calls: `copy_string_to_pstring()`, `PickControlColor()`
- Notes: Dialog prompt is hardcoded "You Must Choose:"

## Control Flow Notes
This file implements widget-layer dialog components called during:
- Dialog setup/initialization (set_labels, set_text, set_value)
- Event handling and user interaction (choose_file, chooseColour, get_text, get_value)
- Screen redraw/refresh (pigDrawer, Draw1Control calls)
- Widget lifecycle (show, hide, activate, deactivate) ΓÇö delegated to parent classes

## External Dependencies
- **Carbon Framework**: `ControlRef`, `MenuRef`, `GetControlPopupMenuHandle()`, `SetControlData()`, `RGBForeColor()`, `PickControlColor()`, etc.
- **Game engine abstractions**: `FileSpecifier`, `player_info`, `NetGetPlayerData()`, `_draw_screen_text()`, `_get_player_color()`
- **Data structures**: `MetaserverPlayerInfo`, `prospective_joiner_info`, `GameListMessage::GameListEntry` (defined elsewhere)
- **Utilities**: `pstring_to_string()`, `copy_string_to_pstring()` (Pascal/C string conversion ΓÇö defined in cseries headers)
- **Boost**: `boost::bind` (in header only)
