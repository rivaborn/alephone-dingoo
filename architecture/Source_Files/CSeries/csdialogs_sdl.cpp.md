# Source_Files/CSeries/csdialogs_sdl.cpp

## File Purpose
SDL-based dialog item manipulation utility providing compatibility wrappers between the Aleph One game engine's SDL dialog system and legacy Mac dialog APIs. Supplies simple functions to query and modify dialog widget properties (text, selection, enabled state) with minimal boilerplate. Created by Woody Zenfell (2001) to ease porting and maintenance across platforms.

## Core Responsibilities
- Hide/show dialog items by toggling widget enabled state
- Extract and insert numeric values into text entry widgets
- Modify widget selection and enabled state with optional parameter handling
- Query current selection/boolean state from dialog controls
- Convert between Pascal-strings (pstring) and C-strings for text field population
- Provide "QQ_" family of defensive functions that gracefully handle null widgets and multiple widget types
- Support dynamic label updates for selector widgets

## Key Types / Data Structures
None defined in this file.

| Name | Kind | Purpose |
|------|------|---------|
| `DialogPtr` | typedef (alias for `dialog*`) | Opaque handle to a dialog; provides widget lookup and state management (defined in sdl_dialogs.h) |
| `widget` | base class | Minimal interface for all dialog controls (defined in sdl_widgets.h) |
| `w_number_entry` | widget subclass | Numeric text input (defined in sdl_widgets.h) |
| `w_select` | widget subclass | Single-selection dropdown/list (defined in sdl_widgets.h) |
| `w_select_popup` | widget subclass | Popup selector, extends w_select_button (defined in sdl_widgets.h) |
| `w_toggle` | widget subclass | Checkbox/boolean toggle (defined in sdl_widgets.h) |
| `w_text_entry` | widget subclass | Free-form text input (defined in sdl_widgets.h) |
| `w_static_text` | widget subclass | Read-only label text (defined in sdl_widgets.h) |

## Global / File-Static State
None.

## Key Functions / Methods

### HideDialogItem
- Signature: `void HideDialogItem(DialogPtr dialog, short item_index)`
- Purpose: Hide a dialog widget by disabling it (SDL cannot truly hide, so disables instead).
- Inputs: `dialog` (owning dialog), `item_index` (numeric widget ID)
- Outputs/Return: None.
- Side effects: Modifies widget enabled state.
- Calls: `dialog->get_widget_by_id()`, `widget::set_enabled()`
- Notes: Silently succeeds if widget not found; semantic gap with Mac version (see file header comment).

### ShowDialogItem
- Signature: `void ShowDialogItem(DialogPtr dialog, short item_index)`
- Purpose: Show a dialog widget by enabling it.
- Inputs: `dialog` (owning dialog), `item_index` (numeric widget ID)
- Outputs/Return: None.
- Side effects: Modifies widget enabled state.
- Calls: `dialog->get_widget_by_id()`, `widget::set_enabled()`
- Notes: Silently succeeds if widget not found.

### extract_number_from_text_item
- Signature: `int32 extract_number_from_text_item(DialogPtr dlg, short item)`
- Purpose: Retrieve numeric value from a w_number_entry widget.
- Inputs: `dlg` (owning dialog), `item` (numeric ID of w_number_entry)
- Outputs/Return: Parsed integer value.
- Side effects: None.
- Calls: `dialog::get_widget_by_id()`, `w_number_entry::get_number()`
- Notes: Asserts if widget is NULL or not a w_number_entry; expects caller to verify widget type.

### insert_number_into_text_item
- Signature: `void insert_number_into_text_item(DialogPtr dlg, short item, int32 number)`
- Purpose: Set numeric value in a w_number_entry widget.
- Inputs: `dlg` (owning dialog), `item` (numeric ID of w_number_entry), `number` (value to set)
- Outputs/Return: None.
- Side effects: Modifies widget state.
- Calls: `dialog::get_widget_by_id()`, `w_number_entry::set_number()`
- Notes: Asserts if widget is NULL or not a w_number_entry.

### modify_selection_control
- Signature: `void modify_selection_control(DialogPtr inDialog, short inWhichItem, short inChangeEnable, short inChangeValue)`
- Purpose: Conditionally enable/disable and set selection on a w_select widget.
- Inputs: `inDialog` (owning dialog), `inWhichItem` (widget ID), `inChangeEnable` (NONE or CONTROL_ACTIVE), `inChangeValue` (1-based selection index or NONE)
- Outputs/Return: None.
- Side effects: Modifies widget enabled state and/or selection.
- Calls: `dialog::get_widget_by_id()`, `widget::set_enabled()`, `w_select::set_selection()`
- Notes: Asserts on NULL widget or type mismatch; converts 1-based index to 0-based for set_selection; NONE constant skips that modification.

### get_selection_control_value
- Signature: `short get_selection_control_value(DialogPtr dialog, short which_control)`
- Purpose: Retrieve current selection from a w_select widget as 1-based index.
- Inputs: `dialog` (owning dialog), `which_control` (widget ID)
- Outputs/Return: 1-based selection index.
- Side effects: None.
- Calls: `dialog::get_widget_by_id()`, `w_select::get_selection()`
- Notes: Asserts on NULL widget or type mismatch; converts 0-based internal index to 1-based for caller.

### QQ_copy_string_from_text_control
- Signature: `const std::string QQ_copy_string_from_text_control(DialogPTR dlg, int item)`
- Purpose: Extract string from a w_text_entry widget, returning std::string (modern C++).
- Inputs: `dlg` (owning dialog), `item` (widget ID)
- Outputs/Return: std::string (empty string if widget not found).
- Side effects: None.
- Calls: `dialog::get_widget_by_id()`, `w_text_entry::get_text()`
- Notes: Gracefully returns empty string on NULL; defensive version of older pstring functions; no assert.

### QQ_set_selector_control_labels
- Signature: `void QQ_set_selector_control_labels(DialogPTR dlg, int item, const std::vector<std::string> labels)`
- Purpose: Dynamically update label options for a w_select_popup widget.
- Inputs: `dlg` (owning dialog), `item` (widget ID), `labels` (vector of new label strings)
- Outputs/Return: None.
- Side effects: Modifies widget label/option list.
- Calls: `dialog::get_widget_by_id()`, `w_select_popup::set_labels()`
- Notes: Silently succeeds if widget not found or is not w_select_popup.

**Notes**: 
Functions prefixed `QQ_` (newer generation) check for NULL before downcasting and return gracefully; older functions (`extract_number_from_text_item`, etc.) assert on type mismatch, expecting the caller to use the correct widget type. Older `copy_pstring_*` functions handle Pascal-string conversion via `a1_p2cstr()` and `a1_c2pstr()` helpers (defined elsewhere in cseries library).

## Control Flow Notes
This file is a stateless utility layer, not part of engine initialization/frame/shutdown. Functions are called on-demand whenever UI code needs to query or modify a live dialog's widgets. Typical usage: interrogate widget state before applying game-state changes, or update dialog appearance in response to game events.

## External Dependencies
- `cseries.h` ΓÇö umbrella header for Aleph One's C-series compatibility library (includes SDL, string/memory utilities, Mac type emulation)
- `sdl_dialogs.h` ΓÇö `dialog` class definition and theme constants
- `sdl_widgets.h` ΓÇö widget class hierarchy (`w_select`, `w_toggle`, `w_text_entry`, etc.)
- C standard library: `strlen()`, `strncpy()`, `strcpy()`, `free()`
- Aleph One utilities (defined elsewhere): `a1_p2cstr()`, `a1_c2pstr()`, `pstrdup()`, `build_stringvector_from_stringset()`
