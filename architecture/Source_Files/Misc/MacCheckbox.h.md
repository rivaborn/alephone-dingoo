# Source_Files/Misc/MacCheckbox.h

## File Purpose
A lightweight C++ wrapper class for macOS checkbox controls. Abstracts native toolbox APIs (ControlHandle, DialogPtr) to provide a simple interface for getting, setting, and toggling checkbox state within a dialog.

## Core Responsibilities
- Wraps a native macOS checkbox control (ControlHandle)
- Manages checkbox state (read, write, toggle operations)
- Associates the checkbox with its dialog item number
- Provides state synchronization with the underlying native control

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| MacCheckbox | class | Wrapper for a single macOS dialog checkbox control |

## Global / File-Static State
None.

## Key Functions / Methods

### GetItem
- Signature: `short GetItem()`
- Purpose: Query which dialog item number this checkbox is bound to
- Inputs: None
- Outputs/Return: Short integer (item ID)
- Side effects: None

### GetState
- Signature: `bool GetState()`
- Purpose: Retrieve current checkbox state (checked or unchecked)
- Inputs: None
- Outputs/Return: Boolean (true = checked)
- Side effects: None
- Calls: `GetControlValue(CHdl)`

### SetState
- Signature: `void SetState(bool State)`
- Purpose: Set the checkbox to a specific state
- Inputs: `State` ΓÇô desired boolean state
- Outputs/Return: None
- Side effects: Updates native control value
- Calls: `SetControlValue(CHdl, State)`

### ToggleState
- Signature: `void ToggleState()`
- Purpose: Flip the checkbox state (checked Γåö unchecked)
- Inputs: None
- Outputs/Return: None
- Side effects: Updates native control value
- Calls: `GetState()`, `SetState()`

### ToggleIfHit
- Signature: `bool ToggleIfHit(short _Item)`
- Purpose: Conditionally toggle state if item ID matches; used for dialog hit testing
- Inputs: `_Item` ΓÇô item ID to compare
- Outputs/Return: Boolean (true if ID matched, false otherwise)
- Side effects: Toggles state only if match occurs
- Calls: `ToggleState()`
- Notes: Idiomatic pattern for macOS dialog event handling

### Constructor
- Signature: `MacCheckbox(DialogPtr DPtr, short _Item, bool State)`
- Purpose: Initialize wrapper with a dialog item reference and initial state
- Inputs: `DPtr` ΓÇô dialog pointer; `_Item` ΓÇô item index; `State` ΓÇô initial state
- Outputs/Return: None (constructed object)
- Side effects: Queries dialog item info, retrieves ControlHandle, applies initial state
- Calls: `GetDialogItem()`, `SetState()`
- Notes: Assumes `_Item` is a valid checkbox control in the dialog

## Control Flow Notes
This is a UI initialization/dialog utility. Typically instantiated when a dialog is created, then used during event handling (hit-testing via `ToggleIfHit`) and state queries.

## External Dependencies
- **macOS Toolbox APIs**: `ControlHandle`, `DialogPtr`, `GetDialogItem()`, `GetControlValue()`, `SetControlValue()` ΓÇô all native macOS dialog/control management
- **License**: GNU General Public License v2+
