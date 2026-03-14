# Source_Files/CSeries/NibsUiHelpers.h

## File Purpose
This header provides RAII utility wrappers and helper functions for managing macOS Carbon/Cocoa UI resources, particularly for Interface Builder (NIB) integration. It abstracts low-level Carbon APIs into high-level automatic resource-cleanup classes and convenience functions for dialog management, event handling, control access, and text I/O.

## Core Responsibilities
- NIB resource loading and automatic window creation with cleanup
- Control (UI widget) reference management and text field access (static/editable, Pascal/C strings)
- Event loop integration: timers, keyboard input, mouse clicks via RAII wrappers
- Modal and sheet dialog execution and lifecycle
- Custom drawing and hit-testing for user-defined controls
- Tab control switching with pane visibility management
- Color picking dialogs and control property getters/setters

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SelfReleasingCFStringRef` | class | RAII wrapper; auto-releases CFString on destruction |
| `AutoNibReference` | class | RAII wrapper for IBNibRef; calls DisposeNibReference in dtor |
| `AutoNibWindow` | class | Loads and owns a window from a NIB; auto-deallocates on scope exit |
| `AutoDrawability` | class | Installs custom drawing callback on a user-pane control |
| `AutoHittability` | class | Installs custom hit-testing callback on a user-pane control |
| `AutoTimer` | class | Event loop timer with optional boost::function callback support |
| `AutoWatcher` | class | Abstract base for event handling; manages EventHandlerUPP |
| `AutoControlWatcher` | class | Watches control-hit events (clicks); fires callback on hit |
| `AutoKeystrokeWatcher` | class | Watches keyboard events (raw key down/repeat); fires callback per character |
| `AutoKeyboardWatcher` | class | Global keyboard watcher; watches specific control for keystrokes |
| `AutoTabHandler` | class | Tab control event watcher; switches active pane visibility on tab click |
| `Modal_Dialog` | class | Simplified modal/sheet dialog runner without handler installation |
| `ParsedControl` | struct | Tuple of (ControlRef, ControlID) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ControlWatcherEvents` | `EventTypeSpec[1]` | static class member | Event filter for control hits; defined elsewhere |
| `KeystrokeWatcherEvents` | `EventTypeSpec[2]` | static class member | Event filter for keyboard input; defined elsewhere |
| `TabControlEvents` | `EventTypeSpec[1]` | static class member | Event filter for tab control; defined elsewhere |

## Key Functions / Methods

### StringToCFString
- Signature: `std::auto_ptr<SelfReleasingCFStringRef> StringToCFString(const std::string& s)`
- Purpose: Convert C++ std::string to Carbon CFString with automatic memory management
- Inputs: `s` (C++ string)
- Outputs/Return: Auto-releasing wrapper around new CFString
- Side effects: Allocates new CFString object
- Calls: (defined in .cpp)
- Notes: Caller owns the auto_ptr; no explicit release needed

### GetCtrlFromWindow
- Signature: `ControlRef GetCtrlFromWindow(WindowRef DlgWindow, uint32 Signature, uint32 ID)`
- Purpose: Fetch a control from a window by signature+ID pair (Inspector Builder IDs)
- Inputs: Window ref, signature, control ID
- Outputs/Return: ControlRef or NULL if not found
- Calls: (Carbon API call, defined in .cpp)

### RunModalDialog / StopModalDialog
- Purpose: Execute/terminate a modal dialog loop (may be a sheet if sheets available)
- RunModalDialog returns true if OK clicked, false otherwise
- Handler called for each control hit with ParsedControl ID and user data
- Special IDs: 1=OK, 2=Cancel

### BuildMenu
- Signature: `void BuildMenu(ControlRef MenuCtrl, bool (*BuildMenuItem)(int, uint8*, bool&, void*), void* BuildMenuData)`
- Purpose: Populate popup menu by callback; callback returns false to stop iteration
- Callback receives: item number (1-indexed), string pointer (output), initial-selection flag (output), user data

### AutoTimer (both constructors)
- Purpose: Install event loop timer with cleanup; supports both function pointers and boost::function<void()>
- Inputs: delay, interval (0 = once-off), handler, optional handler data
- Side effects: Installs handler in current event loop; removes on destruction
- Notes: Boost version uses internal `bounce_boosted_callback` trampoline to invoke stored callback

### Text I/O helpers (GetStaticPascalText, SetStaticCText, etc.)
- Get/Set functions for Pascal and C strings on static or editable text controls
- MaxLen parameter defaults to 255 for safety
- Four variants: (Static/Edit) ├ù (Pascal/C)

### GetCtrlFloatValue / SetCtrlFloatValue
- Convert control values to/from float range [0.0, 1.0]
- Useful for sliders and other ranged controls

## Control Flow Notes
This file provides UI infrastructure rather than game loop integration. It supports:
- **Initialization**: Loading NIBs and creating windows before game start
- **Event handling**: Timers, keyboard, mouse, and control events during runtime
- **Shutdown**: RAII classes clean up on scope exit (no explicit frame step needed)

Modal dialogs block until user input (OK/Cancel). Event watchers are long-lived and must be manually stopped via StopModalDialog or scope exit.

## External Dependencies
- **Carbon/Core Foundation**: CFStringRef, IBNibRef, WindowRef, ControlRef, EventLoopTimer*, EventHandler*, RGBColor, OSType, OSStatus, ControlID, Str255, ConstStr255Param
- **STL**: `<memory>` (auto_ptr), `<string>`, `<vector>`
- **Boost**: `<boost/function.hpp>` (TimerCallback, ControlHitCallback, GotCharacterCallback typedefs)
- **Defined elsewhere**: `initialize_MLTE()`, control text I/O, color picking, menu building, dialog handling (all .cpp implementations)
