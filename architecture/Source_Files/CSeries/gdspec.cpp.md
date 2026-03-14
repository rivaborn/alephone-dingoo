# Source_Files/CSeries/gdspec.cpp

## File Purpose

Manages graphics device (monitor/display) enumeration, configuration, and user selection for the Aleph One game engine. Provides functionality to find compatible displays, build device specifications, set color depth, and present a UI dialog for users to choose their preferred display and color mode.

## Core Responsibilities

- Enumerate and filter available graphics devices based on capability requirements
- Store and match graphics device specifications (resolution, color depth, slot ID)
- Modify graphics device color depth at runtime
- Provide two implementations of a device selection dialog (modern NIB-based and classic Mac OS versions)
- Handle device-to-UI coordinate mapping and scaling for visual display of monitor layout
- Manage platform differences between Carbon and classic Mac OS APIs

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `DevInfo` | struct | (USES_NIBS) Stores device info: control ref, graphics handle, bounds, selection/main status |
| `DesktopDisplayData` | struct | (USES_NIBS) Container for vector of DevInfo structs |
| `PositioningInfo` | struct | (USES_NIBS) Caches scaled position/size data for monitor layout |
| `dev_info` | struct | (non-USES_NIBS) Legacy device info: rect bounds and graphics handle |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `devices` | `dev_info*` | static (non-USES_NIBS) | Dynamically allocated array of device info; freed after dialog closes |
| `devcnt` | `short` | static (non-USES_NIBS) | Count of enumerated devices |
| `mainix` | `short` | static (non-USES_NIBS) | Index of the main graphics device |
| `curix` | `short` | static (non-USES_NIBS) | Index of currently selected device |
| `draw_desktop_upp` | `UserItemUPP` | static (non-USES_NIBS) | UPP for custom desktop drawing routine |
| `draw_group_upp` | `UserItemUPP` | static (non-USES_NIBS) | UPP for group box drawing |
| `device_filter_upp` | `ModalFilterUPP` | static (non-USES_NIBS) | UPP for dialog event filtering |

## Key Functions / Methods

### BestDevice
- **Signature:** `GDHandle BestDevice(GDSpecPtr spec)`
- **Purpose:** Locates the best graphics device matching the given specification (resolution, color depth, optional slot preference).
- **Inputs:** `spec` ΓÇö pointer to GDSpec with desired width, height, bit_depth, flags, and optional slot preference.
- **Outputs/Return:** Returns GDHandle to best matching device; updates `spec->slot` to the matched device's slot.
- **Side effects:** Modifies `spec->slot` if match found.
- **Calls:** `GetDeviceList()`, `GetNextDevice()`, `TestDeviceAttribute()`, `HasDepth()`, `GetSlotFromGDevice()`.
- **Notes:** Prefers exact slot match if available; falls back to first device meeting size/depth requirements. Skips inactive or non-screen devices.

### MatchGDSpec
- **Signature:** `GDHandle MatchGDSpec(GDSpecPtr spec)`
- **Purpose:** Finds the graphics device with a specific slot ID.
- **Inputs:** `spec` ΓÇö pointer to GDSpec with slot field set.
- **Outputs/Return:** Returns GDHandle or NULL if no match.
- **Calls:** `GetDeviceList()`, `GetNextDevice()`, `TestDeviceAttribute()`, `GetSlotFromGDevice()`.

### SetDepthGDSpec
- **Signature:** `void SetDepthGDSpec(GDSpecPtr spec)`
- **Purpose:** Attempts to change the color depth of the device matching the given spec's slot.
- **Inputs:** `spec` ΓÇö pointer to GDSpec with slot and desired bit_depth/flags.
- **Side effects:** Modifies graphics device color depth via `SetDepth()`.
- **Calls:** `GetDeviceList()`, `GetNextDevice()`, `GetSlotFromGDevice()`, `SetDepth()`.
- **Notes:** Early return if device not found or already at requested depth/flags.

### BuildGDSpec
- **Signature:** `void BuildGDSpec(GDSpecPtr spec, GDHandle dev)`
- **Purpose:** Populates a GDSpec structure with the current properties of a graphics device.
- **Inputs:** `dev` ΓÇö graphics device handle to query.
- **Outputs:** Fills `spec` fields: slot, flags, bit_depth, width, height.
- **Calls:** `GetSlotFromGDevice()`, `TestDeviceAttribute()`.

### HasDepthGDSpec
- **Signature:** `bool HasDepthGDSpec(GDSpecPtr spec)`
- **Purpose:** Checks whether the device matching spec's slot supports the requested color depth.
- **Inputs:** `spec` ΓÇö pointer to GDSpec with slot and bit_depth.
- **Outputs/Return:** Returns true if supported, false otherwise.
- **Calls:** `GetDeviceList()`, `GetNextDevice()`, `HasDepth()`.

### EqualGDSpec
- **Signature:** `bool EqualGDSpec(GDSpecPtr spec1, GDSpecPtr spec2)`
- **Purpose:** Compares two GDSpec structures for equality (currently checks slot only).
- **Outputs/Return:** Returns true if slots match.

### GetSlotFromGDevice
- **Signature:** `short GetSlotFromGDevice(GDHandle cur_dev)`
- **Purpose:** Obtains the slot identifier for a graphics device. Under Carbon, counts device enumeration index; on classic Mac OS, accesses hardware slot directly.
- **Inputs:** `cur_dev` ΓÇö graphics device handle.
- **Outputs/Return:** Returns short slot number.
- **Calls:** `GetDeviceList()`, `GetNextDevice()` (Carbon only), `GetDCtlEntry()` (classic Mac).
- **Notes:** Carbon builds slot by enumerating active screen devices; classic Mac accesses opaque hardware slot field via AuxDCE.

### display_device_dialog (USES_NIBS version)
- **Signature:** `void display_device_dialog(GDSpecPtr spec)`
- **Purpose:** Presents a modern NIB-based dialog for selecting a graphics device and color mode. Displays a scaled visual representation of monitor layout with interactive clickable monitors.
- **Inputs:** `spec` ΓÇö pointer to GDSpec with initial slot selection and color mode.
- **Outputs:** Updates `spec->slot` and `spec->flags` based on user choice.
- **Side effects:** Creates/destroys UI controls, shows window/sheet, calls `RunModalDialog()`.
- **Calls:** `AutoNibReference`, `AutoNibWindow`, `GetCtrlFromWindow()`, `CreateUserPaneControl()`, `SetControlID()`, `RunModalDialog()`, device enumeration functions.
- **Notes:** Uses `PositioningInfo` to scale monitor rects into dialog bounds. Attaches `MonitorDrawer` and `MonitorHandler` as callbacks. Supports color/gray radio buttons.

### display_device_dialog (non-USES_NIBS version)
- **Signature:** `void display_device_dialog(GDSpecPtr spec)`
- **Purpose:** Presents a classic Mac OS dialog (loaded from resource template dlgScreenChooser) for device selection.
- **Inputs:** `spec` ΓÇö pointer to GDSpec with initial slot and color mode.
- **Outputs:** Updates `spec->slot` and `spec->flags`.
- **Side effects:** Allocates `devices` array, shows/hides dialog, runs modal event loop with `device_filter_upp`.
- **Calls:** `myGetNewDialog()`, `MapRect()`, `ModalDialog()`, `SetControlValue()`, device enumeration.
- **Notes:** Scales monitor rects to fit dialog desktop area. Uses static array `devices[devcnt]` that is freed after dialog closes. Runs event loop with `device_filter_upp` to handle clicks on monitor representations.

### Helper Functions (noted briefly)
- `MonitorDrawer()` (USES_NIBS): Renders individual monitor representation with selection color overlay and main-device indicator bar.
- `MonitorHandler()` (USES_NIBS): Handles monitor click events, updates selection state, and triggers redraws.
- `draw_desktop()` (non-USES_NIBS): Renders all monitors in classic dialog with theme support.
- `draw_group()` (non-USES_NIBS): Draws group box outline.
- `device_filter()` (non-USES_NIBS): Event filter for dialog; handles mouse clicks on monitor areas.
- `PositioningInfo::PositioningInfo()` (USES_NIBS): Constructor to cache center and size for scaling calculations.

## Control Flow Notes

This file is **dialog/UI layer** ΓÇö called when the user needs to configure display settings. It fits into **initialization/configuration** rather than the game's frame loop:

1. **Device enumeration** occurs during dialog setup via `GetDeviceList()` / `GetNextDevice()` iteration.
2. **User interaction** drives the dialog (event loop via `ModalDialog()` or `RunModalDialog()`).
3. **Device configuration** (color depth) is applied via `SetDepthGDSpec()`, which calls `SetDepth()` to modify hardware state.
4. **Result** is written back to `GDSpec` (slot and flags), which the caller uses to initialize rendering.

The file contains **two implementations** selected at compile time via `#ifdef USES_NIBS`, reflecting the transition from classic Mac OS to Carbon/modern Cocoa environments.

## External Dependencies

- **Mac OS Quickdraw & Graphics Device Manager:** `GDHandle`, `GetDeviceList()`, `GetNextDevice()`, `TestDeviceAttribute()`, `HasDepth()`, `SetDepth()`, `GetSlotFromGDevice()`.
- **Mac OS Dialog Manager:** `DialogPtr`, `GetDialogItem()`, `SetDialogItem()`, `ModalDialog()`, `myGetNewDialog()`, `DisposeDialog()`.
- **Mac OS Control Manager:** `ControlRef`, `ControlHandle`, `SetControlValue()`, `GetControlValue()`, `CreateUserPaneControl()`, `SetControlID()`, `Draw1Control()`, `EmbedControl()`.
- **Mac OS Carbon / Cocoa:** `WindowRef`, `WindowPort`, `IsWindowActive()`, `ShowWindow()`, `HideWindow()`, `ShowSheetWindow()`, `HideSheetWindow()`.
- **CoreFoundation:** `CFStringRef`, `CFSTR()`.
- **Custom CSeries headers:** `cseries.h` (basic types, macros), `shell.h` (game configuration).
- **Custom Mac UI helpers:** `NibsUiHelpers.h` (NIB loading, control helpers, modal dialog framework).
- **Standard library:** `<stdlib.h>`, `<vector>`.
