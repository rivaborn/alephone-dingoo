# Source_Files/Files/wad_prefs_macintosh.cpp

## File Purpose
Implements a Macintosh Carbon preferences dialog that allows users to navigate between different preference sections (e.g., graphics, sound, controls) via a popup menu and configure settings within each section. Remembers the currently selected section across invocations.

## Core Responsibilities
- Create and manage a modal preferences dialog with dynamically loadable sections
- Populate a popup menu with preference section names from resource strings
- Switch between preference sections, handling setup/teardown of UI elements
- Process user interactions: section selection, OK/Cancel, keyboard navigation
- Handle keyboard shortcuts (Page Up/Down, Home/End) for section navigation
- Persist the current section selection across dialog invocations
- Validate teardown operations before dismissing the dialog

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `preferences_dialog_data` | struct | Callbacks and metadata for each preference section (defined in wad_prefs.h) |
| `DialogPtr` | typedef | macOS Carbon dialog handle |
| `ControlHandle` | typedef | macOS Carbon control handle for popup menu |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `current_pref_section` | static short | file-static | Tracks the active preference section index, persisted between dialog invocations |
| `prefInfo` | extern struct preferences_info* | extern | Global pointer to preferences file data (defined elsewhere) |

## Key Functions / Methods

### set_preferences
- **Signature:** `bool set_preferences(struct preferences_dialog_data *funcs, short count, void (*reload_function)(void))`
- **Purpose:** Main entry point; creates and runs the modal preferences dialog with multiple sections.
- **Inputs:**
  - `funcs`: array of preference section descriptors (callbacks, resource IDs, data accessors)
  - `count`: number of preference sections
  - `reload_function`: callback to reload/refresh the game state if user cancels
- **Outputs/Return:** `true` if user clicked OK, `false` if user clicked Cancel
- **Side effects:** Creates a dialog resource (ID 4000), modifies `current_pref_section`, calls section callbacks for setup/teardown
- **Calls:** `myGetNewDialog`, `GetDialogItem`, `GetControlPopupMenuHandle`, `AppendMenu`, `getpstr`, `SetMenuItemText`, `SetControlMaximum`, `handle_switch`, `ShowWindow`, `NewModalFilterUPP`, `ModalDialog`, `GetControlValue`, `DisposeModalFilterUPP`, `DisposeDialog`
- **Notes:** Dialog loop continues until `item_hit <= iCANCEL`; section teardown must return true before switching/closing; handles three casesΓÇösection popup hit, OK button, Cancel button.

### handle_switch
- **Signature:** `static bool handle_switch(DialogPtr dialog, struct preferences_dialog_data *funcs, void **prefs, short new_section, short old_section)`
- **Purpose:** Switch between preference sections, tearing down the old section and setting up the new one.
- **Inputs:**
  - `dialog`: the preferences dialog
  - `funcs`: array of preference descriptors
  - `prefs`: pointer to current section data pointer (will be updated)
  - `new_section`: target section index
  - `old_section`: current section index (NONE if initial setup)
- **Outputs/Return:** `true` if switch succeeded, `false` if teardown of old section failed
- **Side effects:** Removes old DITL items via `ShortenDITL`, loads new DITL via `GetResource`/`AppendDITL`, calls setup callback for new section, updates `*prefs`
- **Calls:** `CountDITL`, `ShortenDITL`, `GetResource`, `AppendDITL`, `ReleaseResource`, section callbacks (`teardown_dialog_func`, `setup_dialog_func`, `get_data`)
- **Notes:** If `old_section == NONE`, skips teardown (initial case). Number of items to remove is `CountDITL(dialog) - NUMBER_VALID_PREF_ITEMS`.

### preferences_filter_proc
- **Signature:** `static pascal Boolean preferences_filter_proc(DialogPtr dialog, EventRecord *event, short *item_hit)`
- **Purpose:** Modal dialog filter that intercepts keyboard events to provide navigation shortcuts (Page Up/Down, Home/End) for cycling through preference sections.
- **Inputs:**
  - `dialog`: the preferences dialog
  - `event`: the incoming event record
  - `item_hit`: output item hit code (modified if keyboard event handled)
- **Outputs/Return:** `true` if event was handled, `false` to pass to default filter
- **Side effects:** Modifies popup control value, sets `*item_hit` to `iPREF_SECTION_POPUP` if keyboard shortcut triggers section change, converts event to `nullEvent`
- **Calls:** `GetDialogItem`, `GetControlValue`, `SetControlValue`, `GetControlMaximum`, `PIN` macro, `general_filter_proc`
- **Notes:** Clamps new value between 1 and `GetControlMaximum()-1`; hardcoded kEND value of 45; only handles keyDown and autoKey events.

## Control Flow Notes
The dialog lifecycle follows a standard macOS modal pattern:
1. **Init**: Create dialog, populate popup menu with section names, call `handle_switch` for initial section (0)
2. **Frame**: Modal loop processes events via filter proc; keyboard shortcuts update popup value and trigger section switches
3. **Section Switch**: `handle_switch` tears down old section (validating), loads new DITL overlay, calls new section's setup callback
4. **Teardown**: On OK/Cancel, calls current section's teardown callback; only closes dialog if teardown returns true; remembers selected section for next invocation

## External Dependencies
- **Notable includes:** `cseries.h` (core macOS/Carbon compatibility), `map.h`, `wad.h` (WAD file structures), `game_errors.h`, `shell.h` (dialog resource constants), `wad_prefs.h` (callback struct definition)
- **External symbols:** `prefInfo` (extern), `myGetNewDialog`, `getpstr`, `ModalDialog`, `AppendMenu`, `SetMenuItemText`, `GetControlPopupMenuHandle`, `GetControlValue`, `SetControlValue`, `SetControlMaximum`, `CountDITL`, `ShortenDITL`, `AppendDITL`, `GetResource`, `ReleaseResource`, `ShowWindow`, `NewModalFilterUPP`, `DisposeModalFilterUPP`, `DisposeDialog`, `GetDialogItem`, `general_filter_proc`, `PIN` macro, dialog constants (`dlogPREFERENCES_DIALOG`, `iPREF_SECTION_POPUP`, `iOK`, `iCANCEL`, `NUMBER_VALID_PREF_ITEMS`)
