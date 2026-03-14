# Source_Files/CSeries/csdialogs.h

## File Purpose
Header file providing cross-platform dialog management abstractions. It bridges Mac OS native dialog APIs and SDL emulations to enable shared dialog code between platforms. Defines type abstractions, macros, and function prototypes for creating, manipulating, and querying dialog controls (text fields, checkboxes, popups, etc.).

## Core Responsibilities
- Define platform-agnostic dialog pointer types (DialogPTR abstracts Mac WindowRef vs. SDL dialog class)
- Declare generic control manipulation API (QQ_* functions) available on all platforms
- Provide common text/number input/output utilities
- Declare Mac OS-specific dialog APIs (window framing, event filtering, control manipulation)
- Declare platform-specific implementations for specialized control types (boolean, selection, enabled/disabled state)
- Provide SDL emulation stubs for Mac OS Toolbox functions (HideDialogItem, ShowDialogItem)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| DialogPTR / DialogPtr | typedef | Platform-dependent dialog pointer: WindowRef (Mac/NIB), dialog* (SDL) |
| dialog | class | SDL dialog container class (forward-declared; implementation elsewhere) |

## Global / File-Static State
None.

## Key Functions / Methods

### QQ_* family (generic control API, all platforms)
- **QQ_control_exists, QQ_set_control_activity, QQ_hide_control, QQ_show_control**: Basic control state/visibility
- **QQ_get_boolean_control_value / QQ_set_boolean_control_value**: Checkbox/toggle state
- **QQ_get_selector_control_value / QQ_set_selector_control_value**: Popup/dropdown selection
- **QQ_set_selector_control_labels / QQ_set_selector_control_labels_from_stringset**: Configure dropdown options
- **QQ_copy_string_from/to_text_control, QQ_extract/insert_number_into_text_control**: Text field I/O

### Common API (both platforms)
- **extract_number_from_text_item / insert_number_into_text_item**: Legacy number I/O from dialog items
- **copy_pstring_from/to_text_field, copy_pstring_to_static_text**: Pascal string manipulation

### Mac OS specific
- **myGetNewDialog**: Create dialog from resource
- **general_filter_proc**: Event filtering callback for modal dialogs
- **modify_control, modify_radio_button_family**: Low-level control state manipulation
- **AdjustRect, get_window_frame**: Window geometry utilities

### Specialized control functions (platform-dependent)
- **modify_selection_control, modify_boolean_control, modify_control_enabled**: Wrapper functions that use Mac OS numbering (1-indexed popups) vs. SDL (0-indexed)
- **get_selection_control_value, get_boolean_control_value**: Accessors with platform-specific behavior

## Control Flow Notes
Not inferable from this header. This is purely an interface definition for dialog management across a game engine initialization and UI interaction layer. The actual frame/render cycle integration is in implementation files.

## External Dependencies
- `<string>`, `<vector>`: Standard C++ containers for label lists
- **Mac OS frameworks** (conditional): Dialogs.h (via SDL_RFORK_HACK), Carbon window/control APIs
- **SDL**: Non-Mac implementations use custom dialog class
- **StringSet resources**: Referenced by ID for label population

## Notes
- Platform abstraction splits implementations via `#ifdef mac` / `#else` / `#endif`
- Comments (ZZZ, jkvw, Br'fin) indicate incremental contributions by multiple developers
- Pascal string (pstring) support suggests legacy Mac OS compatibility
- Numbering scheme mismatch between Mac popups (1-indexed) and SDL controls (0-indexed) motivates the specialized wrapper functions
- HideDialogItem/ShowDialogItem are "approximate SDL emulations" of Mac Toolbox routines (not full parity)
