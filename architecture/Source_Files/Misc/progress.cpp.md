# Source_Files/Misc/progress.cpp

## File Purpose
Manages progress dialog UI for the Aleph One game engine, displaying loading and network operation status. Provides dual implementations: modern Carbon/macOS (NIB-based) and legacy (dialog-based) for backwards compatibility.

## Core Responsibilities
- Create, display, and destroy progress dialogs with native controls
- Update progress bar visuals based on bytes sent/total
- Display contextual messages (map distribution, physics sync, network operations)
- Handle buffer flushing and redraw synchronization
- Support both modern (Carbon/NIB) and legacy (dialog) UI paradigms on macOS

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `progress_data` | struct | Holds dialog reference, saved port, UPP function pointer, and control reference (legacy path only) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ProgressWindow` | `WindowRef` | global (conditionally compiled) | Single active progress window reference; NULL when unused (USES_NIBS only) |
| `progress_data` | `struct progress_data` | static | Dialog state container (legacy non-USES_NIBS path) |
| `Window_Progress` | `CFStringRef` | static const | NIB window identifier string (USES_NIBS only) |

## Key Functions / Methods

### open_progress_dialog
- **Signature:** `void open_progress_dialog(size_t message_id, bool)` (USES_NIBS) / `void open_progress_dialog(size_t message_id)` (legacy)
- **Purpose:** Create and display a progress dialog, initializing message and progress bar
- **Inputs:** `message_id` (string resource ID for the status message); second bool param unused in USES_NIBS
- **Outputs/Return:** None; modifies global `ProgressWindow` or `progress_data`
- **Side effects:** Creates window/dialog, shows UI, sets up progress control, saves/switches GrafPort (legacy)
- **Calls (USES_NIBS):** `CreateWindowFromNib()`, `reset_progress_bar()`, `set_progress_dialog_message()`, `ShowWindow()`, `SetWindowTitleWithCFString()`
- **Calls (legacy):** `GetNewDialog()`, `CreateProgressBarControl()` (Carbon) or `NewUserItemUPP()` (pre-Carbon), `set_progress_dialog_message()`, `ShowWindow()`, `DrawDialog()`, `QDFlushPortBuffer()`
- **Notes:** Asserts dialog creation success; uses feature-test macro to select implementation path

### close_progress_dialog
- **Signature:** `void close_progress_dialog(void)`
- **Purpose:** Hide and deallocate the progress dialog, restore saved graphics port
- **Inputs:** None
- **Outputs/Return:** None; sets `ProgressWindow` to NULL (USES_NIBS) or disposes `progress_data` resources (legacy)
- **Side effects:** Window/dialog destruction, UPP disposal (legacy pre-Carbon), port restoration
- **Calls (USES_NIBS):** `HideWindow()`, `DisposeWindow()`
- **Calls (legacy):** `SetPort()`, `DisposeDialog()`, `DisposeControl()` (Carbon) or `DisposeRoutineDescriptor()` (pre-Carbon)
- **Notes:** Assumes dialog is open; asserts on USES_NIBS variant

### set_progress_dialog_message
- **Signature:** `void set_progress_dialog_message(size_t message_id)`
- **Purpose:** Update the message text displayed in the progress dialog
- **Inputs:** `message_id` (string resource ID; looked up via `getpstr()`)
- **Outputs/Return:** None; modifies dialog text control
- **Side effects:** Text control update, redraw/flush
- **Calls (USES_NIBS):** `getpstr()`, `GetCtrlFromWindow()`, `SetStaticPascalText()`, `Draw1Control()`, `Update()`
- **Calls (legacy):** `GetDialogItem()`, `getpstr()`, `SetDialogItemText()`
- **Notes:** Asserts dialog exists; uses Str255 Pascal string format

### draw_progress_bar
- **Signature:** `void draw_progress_bar(size_t sent, size_t total)`
- **Purpose:** Update progress bar to reflect sent bytes as proportion of total
- **Inputs:** `sent` (bytes transferred), `total` (total bytes)
- **Outputs/Return:** None; modifies progress control value
- **Side effects:** Control value update, port buffer flush
- **Calls (USES_NIBS):** `GetCtrlFromWindow()`, `GetControl32BitMinimum()`, `GetControl32BitMaximum()`, `SetControl32BitValue()`, `Update()`, `QDFlushPortBuffer()`
- **Calls (legacy Carbon):** `GetDialogItem()`, `SetControlValue()`, `QDFlushPortBuffer()`
- **Calls (legacy pre-Carbon):** `GetDialogItem()`, `RGBForeColor()`, `PaintRect()`, `ForeColor()`
- **Notes:** Proportional calculation: `CtrlMin + ((sent * (CtrlMax - CtrlMin)) / total)`; asserts dialog exists; pre-Carbon path manually draws filled rect

### reset_progress_bar
- **Signature:** `void reset_progress_bar(void)`
- **Purpose:** Reset progress bar to empty state (0% progress)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls redraw via `draw_progress_bar(0, 100)` (Carbon) or custom draw (legacy pre-Carbon)
- **Calls (USES_NIBS):** `Update()`
- **Calls (legacy Carbon):** `draw_progress_bar(0, 100)`
- **Calls (legacy pre-Carbon):** `draw_distribute_progress()`
- **Notes:** Asserts dialog exists

### Update (USES_NIBS helper)
- **Signature:** `static void Update()`
- **Purpose:** Flush buffered port to ensure UI updates are visible
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Direct I/O to graphics system
- **Calls:** `QDIsPortBuffered()`, `QDFlushPortBuffer()`
- **Notes:** Only defined under USES_NIBS; guards against unneeded flush on unbuffered ports

### draw_distribute_progress (legacy pre-Carbon custom drawing)
- **Signature:** `static pascal void draw_distribute_progress(DialogPtr dialog, short item_num)`
- **Purpose:** User-item callback to draw a custom progress bar border and background
- **Inputs:** `dialog` (dialog containing item), `item_num` (item ID, typically `iPROGRESS_BAR`)
- **Outputs/Return:** None; modifies dialog drawing
- **Side effects:** QuickDraw graphics operations (frame/fill)
- **Calls:** `GetPort()`, `SetPort()`, `GetWindowPort()`, `GetDialogItem()`, `PenNormal()`, `RGBForeColor()`, `PaintRect()`, `ForeColor()`, `InsetRect()`, `FrameRect()`
- **Notes:** Only compiled for non-Carbon targets; draws gray highlight fill with black frame; saves and restores port

## Control Flow Notes
**Initialization/Display:** Game calls `open_progress_dialog()` when entering a progress phase (network sync, map loading). Dialog remains open during operation. `draw_progress_bar()` is called repeatedly to reflect progress during the operation.

**Shutdown:** Game calls `close_progress_dialog()` when operation completes, cleaning up resources.

**Rendering:** For USES_NIBS, updates are synchronous control value changes with manual buffer flush. For legacy, pre-Carbon path uses user-item drawing callback; Carbon path uses native progress control.

## External Dependencies
- **Notable includes/imports:** `macintosh_cseries.h` (platform macros, memory), `progress.h` (public API), `shell.h` (string resources, UI constants), `NibsUiHelpers.h` (NIB window/control helpers, conditional)
- **Defined elsewhere:** `getpstr()` (string resource loader), `GetCtrlFromWindow()` (control lookup), `SetStaticPascalText()` (text control setter), `CreateWindowFromNib()` (NIB loader), system colors `system_colors` array, resource IDs `strPROGRESS_MESSAGES`, `GUI_Nib`, QuickDraw/Carbon APIs (`QDIsPortBuffered`, `SetControl32BitValue`, etc.)
- **Conditional compilation:** `USES_NIBS` selects modern (Carbon) path; `TARGET_API_MAC_CARBON` gates pre-Carbon vs. Carbon-era legacy code
