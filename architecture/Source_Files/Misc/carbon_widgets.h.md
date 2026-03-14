# Source_Files/Misc/carbon_widgets.h

## File Purpose
Provides C++ wrapper classes for macOS Carbon UI controls used in NIB-based dialogs. Implements a generic data-binding framework (`Bindable<T>` integration) to enable bidirectional synchronization between UI widgets and application state, abstracting away Carbon API complexity.

## Core Responsibilities
- Wrap Carbon control references (ControlRef) in type-safe OOP classes
- Support bidirectional data binding via `Bindable<T>` interface for automatic modelΓåöUI sync
- Manage user interaction callbacks (button hits, control changes, keystroke events) using event watchers
- Provide specialized widgets for common UI patterns: toggles, text fields, file choosers, generic lists, color pickers
- Handle control visibility and activation states
- Encapsulate Carbon API calls and event handler trampolines

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `NIBsControlWidget` | class | Base class for all Carbon control wrappers; stores ControlRef and provides show/hide/activate/deactivate |
| `ToggleWidget` | class | Boolean checkbox control; implements `Bindable<bool>` |
| `SelectorWidget` | class | Dropdown/popup selector; implements `Bindable<int>` for zero-based index selection |
| `ButtonWidget` | class | Clickable button with hit callback |
| `EditTextWidget` | class | Text input field; implements `Bindable<std::string>` |
| `EditNumberWidget` | class | Numeric input field; implements `Bindable<int>` |
| `FileChooserWidget` | class | Composite button + text display for file selection; implements `Bindable<FileSpecifier>` |
| `ListWidget<tElement>` | template class | Generic scrollable DataBrowser list with item selection callbacks; templated on element type |
| `TextboxWidget` | class | Read-only MLTE-based multiline text display (console-like output) |
| `PlayersInGameWidget` | class | Custom-drawn widget for in-game player display using AutoDrawability |
| `ColourPickerWidget` | class | Color swatch with native color picker; implements `Bindable<RGBColor>` |

## Global / File-Static State
None.

## Key Functions / Methods

### SetItems (ListWidget)
- Signature: `void SetItems(const vector<tElement>& items)`
- Purpose: Replace list contents with new items
- Inputs: `items` ΓÇö vector of elements to display
- Outputs/Return: none
- Side effects: Clears old items from DataBrowser, installs new item IDs
- Calls: `ClearItemsFromControl()`, `InstallItemsInControl()`
- Notes: Used to update dynamic lists (players, games) from network messages

### SetupCallbacks (ListWidget)
- Signature: `void SetupCallbacks()` (private)
- Purpose: Install DataBrowser item/notification callbacks and button handler
- Inputs: none
- Outputs/Return: none
- Side effects: Stores `this` pointer in control reference (as SInt32); registers DataBrowser callbacks
- Calls: `SetControlReference()`, `SetDataBrowserCallbacks()`, `boost::bind()`
- Notes: **64-bit hazard**: pointer-as-SInt32 will fail on 64-bit systems; comment flags for redesign

### HandleItemNotification (ListWidget)
- Signature: `void HandleItemNotification(DataBrowserItemID itemId, DataBrowserItemNotification message)` (private)
- Purpose: Dispatch item events (selection, double-click) to application callback
- Inputs: `itemId` ΓÇö affected item; `message` ΓÇö notification type (e.g., kDataBrowserItemDoubleClicked)
- Outputs/Return: none
- Side effects: Invokes `m_itemSelected` callback if double-click detected
- Calls: `ItemForItemId()`, `m_itemSelected(*item, *this)`
- Notes: Called via static trampoline `BounceHandleItemNotification` to convert C callback to method

### FileChooserWidget constructor
- Signature: `FileChooserWidget(ControlRef button_ctrl, ControlRef text_ctrl, Typecode type, const std::string& prompt)`
- Purpose: Initialize composite file chooser (button + filename display)
- Inputs: button/text control refs, `type` (typecode), `prompt` (dialog string)
- Outputs/Return: none
- Side effects: Creates ButtonWidget and StaticTextWidget; installs file-chooser callback via `boost::bind`
- Calls: `new ButtonWidget()`, `new StaticTextWidget()`, `boost::bind(&FileChooserWidget::choose_file, this)`
- Notes: Callback automatically bound; destructor deletes owned widgets

## Control Flow Notes
This is an event-driven UI framework; no render/frame loop. Typical lifecycle: (1) dialog + widgets created from NIB; (2) widgets registered with `BinderSet` for bidirectional sync; (3) Carbon event handlers (managed by AutoWatcher subclasses) fire on user interaction; (4) data propagates via binding callbacks; (5) destructor releases controls.

## External Dependencies
- **cseries.h** ΓÇö Core game types, Carbon/SDL compatibility macros
- **NibsUiHelpers.h** ΓÇö `AutoControlWatcher`, `AutoKeystrokeWatcher`, `AutoDrawability`, `AutoHittability`, string conversion utilities
- **metaserver_messages.h** ΓÇö `MetaserverPlayerInfo`, `GameListMessage` (for list templates)
- **network.h** ΓÇö `prospective_joiner_info` (network player type)
- **tags.h** ΓÇö `Typecode` enum (file type classification)
- **binders.h** ΓÇö `Bindable<T>` template (data binding interface)
- **FileHandler.h** ΓÇö `FileSpecifier` (file path abstraction)
- **Boost** ΓÇö `boost::bind`, `BOOST_STATIC_ASSERT`
- **Carbon API** (macOS) ΓÇö `ControlRef`, `ShowControl`, `SetControl32BitValue`, `TXNObject`, `RGBColor`, `DataBrowserItemID`, etc. (defined elsewhere, used here)
