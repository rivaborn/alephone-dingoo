# Source_Files/Misc/PlayerDialogs.cpp

## File Purpose
Provides modal dialog routines for configuring player-related settings: chase camera position, physics parameters, and crosshair appearance. Supports both modern NIB-based (Carbon) and legacy Mac dialog implementations. Includes validation of numeric inputs and live preview rendering.

## Core Responsibilities
- Display and handle chase camera configuration dialog (position offsets, damping, spring, opacity, void color)
- Display and handle crosshair configuration dialog (thickness, length, shape, color, opacity)
- Validate numeric input from text fields (ranges, stability constraints for chase cam physics)
- Preview crosshair rendering in real-time
- Convert between UI representations and internal data structures
- Handle both NIB-based (modern) and classic Mac dialog event loops

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Configure_ChaseCam_HandlerData` | struct | Holds control references and void color during chase cam dialog interaction |
| `Configure_Crosshair_HandlerData` | struct | Holds control references during crosshair dialog interaction |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `BkgdColor` | `RGBColor` | static | Background color for crosshair preview; shared across dialog sessions |
| `FLOAT_WORLD_ONE` | `const float` | global | Constant for converting world distance units to/from floats |

## Key Functions / Methods

### Configure_ChaseCam
- **Signature:** `bool Configure_ChaseCam(ChaseCamData &Data)`
- **Purpose:** Main entry point for chase camera configuration dialog; allows user to adjust position offsets, physics parameters, and void color rendering.
- **Inputs:** Reference to `ChaseCamData` structure with initial values
- **Outputs/Return:** `true` if user clicked OK (data modified); `false` if cancelled (data unchanged)
- **Side effects:** Modifies `Data` structure, may modify global OpenGL void color from `Get_OGL_ConfigureData()`; displays modal dialog blocking further interaction
- **Calls (NIB version):** `RunModalDialog`, `Configure_ChaseCam_Handler`, `VerifyAndAdjust`, `GetCtrlFromWindow`, `SetFloat`, `GetFloat`, `FloatRoundoff`; (Legacy version) `ModalDialog`, `myGetNewDialog`, `GetDialogItem`, `SetFloat`, `GetColor`, `SysBeep`
- **Notes:** Two implementations: `#ifdef USES_NIBS` for modern Carbon, else classic Mac API. Validates damping factor in [-1, 1], spring stability (prevents oscillatory/overdamped instability), and opacity in [0, 1]. If validation fails, beeps and allows user to correct. Converts world units (WORLD_ONE) to/from float for display.

### Configure_Crosshairs
- **Signature:** `bool Configure_Crosshairs(CrosshairData &Data)`
- **Purpose:** Main entry point for crosshair configuration dialog; allows user to adjust appearance (thickness, length, shape, color) and preview in real-time.
- **Inputs:** Reference to `CrosshairData` structure with initial values
- **Outputs/Return:** `true` if user clicked OK (data modified); `false` if cancelled (data unchanged)
- **Side effects:** Modifies `Data` structure; temporarily activates crosshairs for preview; displays modal dialog; may modify global `BkgdColor` if user picks background color
- **Calls (NIB version):** `RunModalDialog`, `Configure_Crosshair_Handler`, `GetCtrlFromWindow`, `SetShort`, `SetFloat`, `GetShort`, `GetFloat`, `Crosshairs_SetActive`, `GetCrosshairData`; (Legacy version) `ModalDialog`, `myGetNewDialog`, `DoPreview`, `GetColor`, `DrawDialog`, `SysBeep`
- **Notes:** Two implementations. Saves/restores old crosshair state so preview doesn't affect game rendering. Preview shows crosshair in context of background color. Validates thickness > 0, FromCenter ΓëÑ 0, Length > 0, opacity in [0, 1]; rejects invalid and triggers beep with reversion.

### Configure_ChaseCam_HandlerData::VerifyAndAdjust
- **Signature:** `bool VerifyAndAdjust()`
- **Purpose:** Validates and adjusts chase cam parameters to ensure physical stability; corrects out-of-range values.
- **Inputs:** None (reads from member control references)
- **Outputs/Return:** `true` if all values valid; `false` if any corrections were made
- **Side effects:** Updates control text fields if values out of range; triggers system beep if corrections made
- **Calls:** `GetFloat`, `SetFloat`, `PIN` macro, `fabs`, `sqrt`
- **Notes:** Validates damping Γêê [-1, 1], spring stability (prevents oscillatory collapse when Damp┬▓ + Spring ΓëÑ 1, prevents overdamped divergence), opacity Γêê [0, 1]. Uses PIN macro to clamp values. For spring validation: if Spring ΓëÑ 0, ensures Damp┬▓ + Spring < 1; if Spring < 0 (overdamped), ensures |Damp| + ΓêÜ(-Spring) < 1.

### Configure_ChaseCam_Handler / Configure_Crosshair_Handler
- **Signature:** `static void Configure_ChaseCam_Handler(ParsedControl &Ctrl, void *Data)` / `static void Configure_Crosshair_Handler(ParsedControl &Ctrl, void *Data)`
- **Purpose:** Event handlers for NIB-based dialogs; respond to control interactions (color picker, verify button, shape/preview updates).
- **Inputs:** `ParsedControl` event, opaque `Data` pointer (cast to handler data struct)
- **Outputs/Return:** void; side effects on dialog state
- **Side effects:** May call color picker, update crosshair rendering, trigger validation
- **Calls:** `VerifyAndAdjust`, `PickControlColor`, `GetShort`, `GetFloat`, `Crosshairs_Render`, `Draw1Control`, `GetControl32BitValue`
- **Notes:** Chase cam handler: responds to color selection and verify button. Crosshair handler: updates crosshair data from controls and redraws preview on shape/preview change; opens color pickers for crosshair and background.

### PreviewDrawer / DoPreview
- **Signature:** `void PreviewDrawer(ControlRef Ctrl, void *Data)` (NIB) / `static pascal void DoPreview(DialogPtr Dialog, short ItemNo)` (Legacy)
- **Purpose:** Render callback for crosshair preview control; draws background and crosshairs in bounded rectangle.
- **Inputs:** Control reference (or dialog + item number); data pointer (unused in NIB version)
- **Outputs/Return:** void; draws to current graphics context
- **Side effects:** Modifies graphics context (pen, color, clipping)
- **Calls:** `GetControlBounds`, `PenNormal`, `RGBForeColor`, `PaintRect`, `ClipRect`, `Crosshairs_Render`, `FrameRect`, `RGBBackColor` (legacy)
- **Notes:** NIB version simpler (assumes context set up); legacy version saves/restores pen state and colors. Both clip rendering to preview rectangle and restore graphics state. Temporarily activates crosshairs for rendering.

## Control Flow Notes
- **Dialog initialization:** Create dialog (NIB or resource), retrieve all controls, populate with current `Data` values, show window as sheet or modal dialog.
- **User interaction:** In NIB mode, `RunModalDialog` dispatches to handler callbacks on control hits. In legacy mode, manual `ModalDialog` event loop processes OK/Cancel/Color/Preview/checkbox hits.
- **Validation:** On OK or Preview (legacy), validate all numeric fields. If invalid, beep, revert bad values to display, and continue loop (legacy) or return false (NIB).
- **Completion:** On OK, copy validated values from UI controls back to `Data` structure. On Cancel, discard changes (legacy saves original, restores if !IsOK). Hide and dispose dialog.

## External Dependencies
- **Includes:** `ChaseCam.h` (ChaseCamData, chase cam state functions), `Crosshairs.h` (CrosshairData, crosshair state/render), `OGL_Setup.h` (OGL_ConfigureData, Get_OGL_ConfigureData()), `MacCheckbox.h` (MacCheckbox class), `interface.h`, `world.h`, `shell.h`
- **Defined elsewhere:** `GetCtrlFromWindow`, `SetEditCText`, `GetEditCText`, `PickControlColor`, `RunModalDialog`, `AutoNibWindow`, `AutoNibReference`, `AutoDrawability`, `AutoHittability`, `SwatchDrawer` (NIB helpers); `myGetNewDialog`, `GetDialogItem`, `SetDialogItem`, `ModalDialog`, `ShowSheetWindow`, `HideSheetWindow`, `GetDialogWindow`, `DisposeDialog`, `GetColor`, `DrawDialog`, `SetThemeWindowBackground` (legacy Mac APIs); `GetCrosshairData`, `Crosshairs_IsActive`, `Crosshairs_SetActive`, `Crosshairs_Render`, `Get_OGL_ConfigureData` (game engine)
- **Macros:** `TEST_FLAG`, `SET_FLAG`, `PIN`, `NORMALIZE_ANGLE` (from cseries.h)
