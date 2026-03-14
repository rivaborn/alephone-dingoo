# Source_Files/CSeries/csdialogs_macintosh.cpp

## File Purpose

Provides macOS dialog management and control manipulation for the Aleph One game engine. Wraps Mac OS Toolbox and Carbon APIs to abstract dialog creation, event handling, control manipulation, and modal dialog orchestration. Supports both classic Mac DITL-based dialogs and modern NIB-based interfaces.

## Core Responsibilities

- Dialog creation and initialization with standard button setup
- Control value extraction/insertion (numeric and text operations)
- Control state modification (hiliting, enabling/disabling, value setting)
- Dialog event filtering and window update handling
- User item frame drawing (region-based on classic, theme-based on Carbon)
- Modal dialog execution with event handler dispatch
- Popup menu population via callbacks
- Color picker integration with live preview
- Tab control pane visibility management
- Timer and event watcher lifecycle management
- Cross-platform control abstraction (QQ_* functions)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| ModalDialogHandlerData | struct | Stores dialog window, sheet flag, handler, and result state |
| ControlDrawerData | struct | Associates draw function pointer with control-specific data |
| ParsedControl | struct | Bundles ControlRef with ControlID for event dispatch |
| AutoNibWindow | class | RAII wrapper: loads window from NIB, disposes on destruction |
| AutoDrawability | class | RAII wrapper: installs/disposes control drawing procedure |
| AutoHittability | class | RAII wrapper: installs/disposes control hit-test procedure |
| AutoTimer | class | RAII wrapper: manages event-loop timer lifecycle |
| AutoWatcher | class | Base class for event handler watchers (abstract) |
| AutoControlWatcher | class | Watches control hit events, invokes callback |
| AutoKeystrokeWatcher | class | Watches keyboard events on controls |
| AutoKeyboardWatcher | class | Legacy keyboard event handler wrapper |
| AutoTabHandler | class | Manages tab control: shows active pane, hides others |
| Modal_Dialog | class | Modern modal dialog lifecycle manager |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| cursor_tracking | bool | static | Flag to enable dialog cursor tracking in filter proc |
| header_proc | dialog_header_proc_ptr | static | Callback for custom dialog header drawing |
| third | RGBColor | static const | Gray color (0x5555, 0x5555, 0x5555) for region painting |
| general_filter_upp | ModalFilterUPP | static/global | UPP for general_filter_proc (compatibility wrapper) |
| global_signature | OSType | static | Current control signature for lookup in QQ_* functions |
| PickCtrl | ControlRef | static | Control being colored in active color picker session |
| PickClrPtr | RGBColor* | static | Pointer to color being edited (for live updates) |

## Key Functions / Methods

### myGetNewDialog
- Signature: `DialogPtr myGetNewDialog(short id, void *storage, WindowPtr before, long refcon)`
- Purpose: Wrapper around GetNewDialog that configures default/cancel buttons and stores refcon
- Inputs: Dialog resource ID, dialog storage, window ordering, reference constant
- Outputs/Return: DialogPtr (dialog reference or NULL)
- Side effects: Sets dialog button defaults, stores refcon in window
- Calls: GetNewDialog, SetWRefCon, GetDialogWindow, GetDialogItem, SetDialogDefaultItem, SetDialogCancelItem
- Notes: Assumes iOK=1, iCANCEL=2; handles null dialog gracefully

### frame_useritems
- Signature: `static void frame_useritems(DialogPtr dlg)`
- Purpose: Draw frames/separators around user-item controls in dialog
- Inputs: DialogPtr
- Outputs/Return: void
- Side effects: Modifies GrafPort, draws to dialog
- Calls: CountDITL, GetDialogItem, RectRgn, UnionRgn, DiffRgn, DisposeRgn; Carbon: DrawThemeSeparator, IsWindowActive; Classic: PaintRgn
- Notes: Carbon uses theme separators at top of group; classic uses 1-pixel region frames. Complex region boolean ops to avoid painting over other controls.

### general_filter_proc
- Signature: `pascal Boolean general_filter_proc(DialogPtr dlg, EventRecord *event, short *hit)`
- Purpose: Universal event filter for dialogs; dispatches updates and activations
- Inputs: DialogPtr, EventRecord, hit item output pointer
- Outputs/Return: Boolean (whether event was handled)
- Side effects: Port switching, calls update_any_window or frame_useritems, may call activate_any_window
- Calls: GetPort, SetPort, SetDialogTracksCursor, GetWindowPort, GetWindowKind, GetPortBounds, GetDialogWindow, GetDialogFromWindow, LocalToGlobal, StdFilterProc
- Notes: Delegates non-dialog windows to external handlers; always returns StdFilterProc result

### modify_control
- Signature: `void modify_control(DialogPtr dlg, short item, short hilite, short value)`
- Purpose: Modify control appearance (hilite state) and/or numeric value
- Inputs: Dialog, item index, hilite state (NONE to skip), new value (NONE to skip)
- Outputs/Return: void
- Side effects: Control state changes
- Calls: GetDialogItem, HiliteControl, SetControlValue; Carbon: GetDialogItemAsControl, DisableControl, DeactivateControl, EnableControl, ActivateControl (depending on __MACH__)
- Notes: Carbon prefers control hierarchy enable/disable; falls back to HiliteControl on classic or if hierarchy unavailable

### modify_radio_button_family
- Signature: `void modify_radio_button_family(DialogPtr dlg, short firstItem, short lastItem, short activeItem)`
- Purpose: Set one radio button in a group to checked, others to unchecked
- Inputs: Dialog, first/last item in group, item to activate
- Outputs/Return: void
- Side effects: Sets control values
- Calls: GetDialogItem (loop), SetControlValue (loop)
- Notes: Trivial iteration over item range; no state validation

### extract_number_from_text_item / insert_number_into_text_item
- Signature: `int32 extract_number_from_text_item(DialogPtr dlg, short item)` / `void insert_number_into_text_item(DialogPtr dlg, short item, int32 number)`
- Purpose: Convert between numeric and text representations in dialog items
- Inputs: Dialog, item index; number (for insert)
- Outputs/Return: int32 (extracted number) / void
- Side effects: Text field updates
- Calls: get_handle_for_dialog_item, GetDialogItemText, SetDialogItemText, StringToNum, NumToString
- Notes: Uses Mac Str255 (Pascal string) intermediates

### hit_dialog_button
- Signature: `bool hit_dialog_button(DialogPtr dlg, short item)`
- Purpose: Simulate button press (hilite, delay, unhilite)
- Inputs: Dialog, button item
- Outputs/Return: bool (true if button was unhilited, false if already hilited)
- Side effects: Control hiliting, brief delay (8 ticks)
- Calls: GetDialogItem, GetControlHilite, HiliteControl, Delay
- Notes: Returns false if button already pressed; no actual action taken

### get_handle_for_dialog_item
- Signature: `static OSErr get_handle_for_dialog_item(DialogPtr __dialog, short __item, Handle *__handle)`
- Purpose: Get control handle, preferring modern control embedding over legacy DITL
- Inputs: DialogPtr, item index, output handle pointer
- Outputs/Return: OSErr (noErr)
- Side effects: None
- Calls: GetDialogItemAsControl (tries first), GetDialogItem (fallback)
- Notes: Transparently upgrades to ControlRef if control hierarchy available

### RunModalDialog
- Signature: `bool RunModalDialog(WindowRef DlgWindow, bool IsSheet, void (*DlgHandler)(ParsedControl &, void *), void *DlgData)`
- Purpose: Block and run a modal dialog with event dispatch
- Inputs: Window, sheet flag, optional event handler, handler user data
- Outputs/Return: bool (true if OK pressed, false if Cancel)
- Side effects: Shows window, runs app modal loop, hides window, installs/removes event handler
- Calls: NewEventHandlerUPP, InstallWindowEventHandler, ShowSheetWindow, ShowWindow, RunAppModalLoopForWindow, DisposeEventHandlerUPP, conditional HideSheetWindow
- Notes: Dispatch loops on kEventControlHit; iOK/iCANCEL with signature 0 exit dialog

### BuildMenu
- Signature: `void BuildMenu(ControlRef MenuCtrl, bool (*BuildMenuItem)(int, uint8*, bool&, void*), void *BuildMenuData)`
- Purpose: Populate popup menu from callback; tracks initial selection
- Inputs: Popup control, builder callback, user data
- Outputs/Return: void
- Side effects: Menu structure modified
- Calls: GetControlPopupMenuHandle, CountMenuItems, DeleteMenuItem, AppendMenu, SetMenuItemText, SetControl32BitMaximum, SetControl32BitValue
- Notes: Callback returns false to stop; initial item defaults to 1

### QQ_copy_string_from_text_control / QQ_copy_string_to_text_control
- Signature: `const std::string QQ_copy_string_from_text_control(DialogPTR dlg, int item)` / `void QQ_copy_string_to_text_control(DialogPTR dlg, int item, const std::string &s)`
- Purpose: Extract/insert std::string from/to control (handles both static and edit text)
- Inputs: Dialog, item, string (for insert)
- Outputs/Return: std::string (extracted) / void
- Side effects: Control text modified (insert only); redraw requested
- Calls: get_control_from_window, GetControlKind, GetControlDataSize, GetControlData, SetControlData, Draw1Control
- Notes: Distinguishes kControlKindEditText and kControlKindStaticText; returns empty string on not found

### PickControlColor
- Signature: `bool PickControlColor(ControlRef Ctrl, RGBColor *ClrPtr, ConstStr255Param Prompt)`
- Purpose: Launch system color picker; update control color with live preview
- Inputs: Control, color pointer, prompt string
- Outputs/Return: bool (true if color accepted)
- Side effects: Updates color object, triggers control redraw, sets globals for live callback
- Calls: NewColorChangedUPP, PickColor, DisposeColorChangedUPP, Draw1Control
- Notes: Uses globals (PickCtrl, PickClrPtr) for live update from PickColorChangedFunc; reverts on cancel

### AutoTimer
- Signature: Constructor: `AutoTimer(EventTimerInterval Delay, EventTimerInterval Interval, EventLoopTimerProcPtr Handler, void *HandlerData)` and overload with TimerCallback
- Purpose: RAII wrapper for event-loop timers
- Inputs: Delay, interval, handler (UPP or boost::function), optional user data
- Outputs/Return: void
- Side effects: Installs timer on current event loop; removes on destruction
- Calls: GetCurrentEventLoop, NewEventLoopTimerUPP, InstallEventLoopTimer, RemoveEventLoopTimer, DisposeEventLoopTimerUPP
- Notes: Two constructors: one for raw UPP, one for boost::function with bounce callback

### AutoWatcher / AutoControlWatcher / AutoKeystrokeWatcher
- Signature: `AutoWatcher::AutoWatcher(ControlRef ctrl, int num_event_types, const EventTypeSpec* event_types)` (protected base); public subclasses
- Purpose: Base class for control event watchers; subclasses override act() to handle events
- Inputs: Control, event type spec count/array
- Outputs/Return: void
- Side effects: Installs control event handler
- Calls: NewEventHandlerUPP, InstallControlEventHandler, DisposeEventHandlerUPP (base dtor)
- Notes: Static callback trampolines to virtual act(); subclasses define event specs and act() logic

### Modal_Dialog
- Signature: Constructor: `Modal_Dialog(WindowRef dialogWindow, bool isSheet)` / Methods: `bool Run()`, `void Stop(bool result)`
- Purpose: Modern modal dialog manager without built-in event handler
- Inputs: Window, sheet flag; result for Stop()
- Outputs/Return: bool (result passed to Stop)
- Side effects: Shows/hides window, runs/quits modal loop
- Calls: SetThemeWindowBackground, ShowSheetWindow, ShowWindow, RunAppModalLoopForWindow, QuitAppModalLoopForWindow, HideSheetWindow, HideWindow
- Notes: Caller responsible for installing event handlers on controls; cleaner than RunModalDialog for modern code

## Control Flow Notes

**Initialization Phase:**
- Dialog created via `myGetNewDialog()` or `AutoNibWindow` (NIB-based)
- Optional event filter installed via `get_general_filter_upp()`
- Controls configured via `modify_control()`, menus via `BuildMenu()`

**Event/Update Phase:**
- `general_filter_proc` intercepts updateEvt and activateEvt
- frame_useritems redraws user item frames on each update
- Control events dispatched through ModalDialogHandler or custom event handlers

**Modal Loop:**
- `RunModalDialog()` or `Modal_Dialog::Run()` blocks on app modal loop
- iOK/iCANCEL buttons trigger `StopModalDialog()` and return result

**Shutdown:**
- Dialog windows disposed automatically (`AutoNibWindow` dtor) or explicitly
- Timers, watchers, handlers cleaned up via RAII destructors

## External Dependencies

- **Includes (notable):**
  - `<Carbon/Carbon.h>` ΓÇô Mac OS Toolbox & Carbon APIs
  - `"csdialogs.h"` ΓÇô Dialog interface declarations
  - `"NibsUiHelpers.h"` ΓÇô NIB-specific helpers (AutoNibWindow, control text accessors)
  - `<string>`, `<vector>` ΓÇô STL containers for modern string/menu APIs

- **External symbols (defined elsewhere):**
  - `update_any_window()`, `activate_any_window()` ΓÇô Window event dispatchers
  - `build_stringvector_from_stringset()`, `pstring_to_string()`, `copy_string_to_pstring()` ΓÇô String utilities
  - `csprintf()`, `temporary`, `ptemporary` ΓÇô Debug sprintf
  - `vassert()` ΓÇô Debug assertion with message
