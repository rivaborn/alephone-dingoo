# Source_Files/Misc/OGL_Dialog.cpp

## File Purpose
Provides dialog boxes for configuring OpenGL rendering options in Aleph One. Supports both modern Carbon NIB-based dialogs and legacy classic Mac dialogs, allowing users to customize texture quality, rendering flags (Z-buffer, fog, 3D models, etc.), and color schemes for void and landscapes.

## Core Responsibilities
- Display and handle texture configuration dialogs (walls, landscapes, inhabitants, weapons)
- Display and handle main OpenGL configuration dialog with all rendering options
- Manage checkbox states for 11+ rendering feature flags (Z-buffer, fog, 3D models, etc.)
- Handle color picking for void color and 8 landscape color swatches (4 landscapes ├ù 2 channels)
- Support both NIB-based (modern/Carbon) and legacy classic Mac dialog implementations
- Manage anisotropy slider with log-linear scaling and texture quality popup menus
- Synchronize temporary dialog state with persistent OGL_ConfigureData structure

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `TextureConfigDlgHandlerData` | struct | Holds references to texture config controls and the texture list for dialog handler callbacks |
| `OGL_Dialog_Handler_Data` | struct | Holds temporary color and texture configuration data while main dialog is open |
| `OGL_Texture_Configure` | struct (external) | Single texture type's settings (filter, resolution, color depth, max size) |
| `OGL_ConfigureData` | struct (external) | Complete OpenGL configuration including all textures, flags, colors, and quality settings |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `TxtrConfigList` | `OGL_Texture_Configure[OGL_NUMBER_OF_TEXTURE_TYPES]` | static (non-NIB path only) | Local copy of texture configurations during dialog session |
| `VoidColor` | `RGBColor` | static (non-NIB path only) | Temporary void color during dialogΓÇöpersists across dialog calls to support cancel |
| `LscpColors` | `RGBColor[4][2]` | static (non-NIB path only) | Temporary landscape colors (4 types ├ù 2 channels: ground/sky) |
| `CheckboxDispatch` | `int[11][2]` | static | Lookup table mapping dialog item IDs to OGL flag bits |

## Key Functions / Methods

### TextureConfigDlgHandler
- **Signature:** `static void TextureConfigDlgHandler(ParsedControl &Ctrl, void *Data)`
- **Purpose:** Event handler for texture configuration dialog (NIB/Carbon path). Updates filter/resolution/color controls when "Based On" menu changes.
- **Inputs:** Parsed control (item ID + reference), handler data with texture list
- **Outputs/Return:** None (void)
- **Side effects:** Modifies control values (`NearCtrl`, `FarCtrl`, `ResCtrl`, `ColorCtrl`) to reflect selected source texture
- **Calls:** `GetControl32BitValue`, `SetControl32BitValue`
- **Notes:** Only acts if "Based On" menu item (`BasedOn_Item`) was hit and source differs from current texture

### TextureConfigureDialog (NIB version)
- **Signature:** `static bool TextureConfigureDialog(OGL_Texture_Configure *TxtrConfigList, short WhichTexture)`
- **Purpose:** Modal dialog for configuring a single texture type (near/far filters, resolution, color format)
- **Inputs:** Texture config array and index of texture to edit
- **Outputs/Return:** `true` if OK pressed, `false` if canceled
- **Side effects:** Creates and displays modal dialog window; modifies `TxtrConfigList[WhichTexture]` on OK
- **Calls:** `AutoNibReference`, `AutoNibWindow`, `GetCtrlFromWindow`, `SetControl32BitValue`, `GetControlPopupMenuHandle`, `GetMenuItemText`, `SetStaticPascalText`, `RunModalDialog`
- **Notes:** Uses RAII wrappers (`AutoNibReference`, `AutoNibWindow`) for automatic resource cleanup; swaps filter/resolution indices (+1 for 1-indexed menus)

### TextureConfigureDialog (legacy version)
- **Signature:** `static bool TextureConfigureDialog(short WhichTexture)`
- **Purpose:** Modal dialog for texture configuration in classic Mac environment. Copies from another texture type if user selects "Based On" option.
- **Inputs:** Index of texture to edit
- **Outputs/Return:** `true` if OK, `false` if canceled
- **Side effects:** Modifies static `TxtrConfigList[WhichTexture]` on OK; manages dialog visibility and focus
- **Calls:** `myGetNewDialog`, `GetDialogItem`, `SetControlValue`, `GetControlValue`, `GetControlPopupMenuHandle`, `GetMenuItemText`, `SetDialogItemText`, `ModalDialog`, `BringToFront`, `ActiveNonFloatingWindow`, `SelectWindow`, `ShowWindow`, `HideWindow`, `DisposeDialog`
- **Notes:** Manual event loop; saves/restores front window for sheet support; swaps window to front if `USE_SHEETS` defined

### OGL_Dialog_Handler
- **Signature:** `static void OGL_Dialog_Handler(ParsedControl &Ctrl, void *Data)`
- **Purpose:** Event handler for main OpenGL dialog (NIB/Carbon path). Dispatches to texture dialog, color picker, or stores color changes.
- **Inputs:** Parsed control, handler data with colors and textures
- **Outputs/Return:** None (void)
- **Side effects:** Calls `TextureConfigureDialog` on texture buttons; modifies color references via `PickControlColor`
- **Calls:** `TextureConfigureDialog`, `PickControlColor`, `getpstr`
- **Notes:** Handles color swatch controls (void + 8 landscape swatches) and 4 texture configuration buttons

### OGL_ConfigureDialog (NIB version)
- **Signature:** `bool OGL_ConfigureDialog(OGL_ConfigureData& Data)`
- **Purpose:** Main OpenGL configuration dialog. Returns `true` if user confirmed changes, `false` if canceled.
- **Inputs:** Reference to config data structure to edit
- **Outputs/Return:** `true` on OK, `false` on cancel; modifies `Data` in-place on OK
- **Side effects:** Creates modal dialog; initializes swatches with custom draw/hit handlers; decodes anisotropy slider and texture quality popups from UI state back to config values
- **Calls:** `AutoNibReference`, `AutoNibWindow`, `GetCtrlFromWindow`, `SetControl32BitValue`, `AutoDrawability`, `AutoHittability`, `SwatchDrawer`, `RunModalDialog`, `GetCtrlFloatValue`, `SetCtrlFloatValue`, `GetControlValue`, `SetControlValue`
- **Notes:** Anisotropy uses half-linear, half-log mapping (0ΓÇô1 linear, 1ΓÇô5 logarithmic); wall quality 1ΓÇô5 maps to sizes {0, 128, 256, 512, 1024}; landscape quality 1ΓÇô5 maps to {0, 256, 512, 1024, 2048}; uses `PIN()` macro to clamp values

### OGL_ConfigureDialog (legacy version)
- **Signature:** `bool OGL_ConfigureDialog(OGL_ConfigureData& Data)`
- **Purpose:** Main OpenGL configuration dialog (classic Mac path). Manages checkboxes, color picking, and texture dialog access.
- **Inputs:** Reference to config data
- **Outputs/Return:** `true` on OK, `false` on cancel; modifies `Data` in-place on OK
- **Side effects:** Creates dialog; sets up custom draw UPP for swatches; manages checkbox objects; modifies static color/texture copies on OK
- **Calls:** `myGetNewDialog`, `GetDialogItem`, `MacCheckbox` constructor, `NewUserItemUPP`, `SetDialogItem`, `TextureConfigureDialog`, `GetColor`, `DrawDialog`, `SetThemeWindowBackground`, `ShowSheetWindow`, `SelectWindow`, `ShowWindow`, `ModalDialog`, `HideWindow`, `DisposeDialog`, `DisposeUserItemUPP`
- **Notes:** Manual event loop with sheet support (`USE_SHEETS`); creates 11 `MacCheckbox` objects to manage toggle flags; applies `PaintSwatch` UPP to all color swatches

### PaintSwatch (legacy version only)
- **Signature:** `static pascal void PaintSwatch(DialogPtr DPtr, short ItemNo)`
- **Purpose:** Custom USERITEM draw procedure for color swatches. Fills a control with its color and frames it.
- **Inputs:** Dialog pointer and item number
- **Outputs/Return:** None (void); draws to screen
- **Side effects:** Saves/restores pen state, foreground/background colors; draws filled rectangle in color then black frame
- **Calls:** `GetDialogItem`, `GetPenState`, `GetBackColor`, `GetForeColor`, `PenNormal`, `RGBForeColor`, `PaintRect`, `ForeColor`, `FrameRect`, `SetPenState`, `RGBBackColor`, `RGBForeColor`
- **Notes:** Searches static `VoidColor` and `LscpColors` arrays to find the color pointer for the given item ID

## Control Flow Notes
Entry point is `OGL_ConfigureDialog()` (public), which is called from preferences UI. 

**NIB path** (`USES_NIBS` defined):
1. Opens main dialog via `AutoNibWindow`
2. Initializes all controls from `Data`
3. Sets up drawable/hittable swatches with `AutoDrawability`/`AutoHittability`
4. Calls `RunModalDialog()` with handler, which returns on OK/Cancel
5. If OK, writes all control values back to `Data`, including anisotropy decode and quality popup decode

**Legacy path**:
1. Creates dialog via `myGetNewDialog`
2. Initializes `MacCheckbox` objects for 11 flags
3. Sets up custom `PaintSwatch` UPP for color swatches
4. Enters manual `ModalDialog` loop, dispatching to texture/color dialogs on button hits
5. If OK, collects all checkbox and color states and writes back to `Data`

Both paths preserve existing `Data` on cancel by storing temporaries separately (`HandlerData` struct in NIB, static variables in legacy).

## External Dependencies
- **Includes:** `cseries.h`, `shell.h`, `OGL_Setup.h`, `MacCheckbox.h`, `NibsUiHelpers.h` (if `USES_NIBS`)
- **External symbols used:** `myGetNewDialog`, `ModalDialog`, `GetDialogItem`, `SetDialogItem`, `GetColor`, `GetControlPopupMenuHandle`, `GetMenuItemText`, `SetDialogItemText`, `ActiveNonFloatingWindow`, `GetDialogWindow`, `TEST_FLAG`, `SET_FLAG`, `PIN()`, `getpstr`, `ptemporary`, `NewUserItemUPP`, `DisposeUserItemUPP`, `RGBColor`, `RGBForeColor`, `RGBBackColor`, `ControlHandle`, `ControlRef`, `DialogPtr`, `WindowRef`, `MenuRef`, `MenuHandle`
- **Macros/constants from headers:** Dialog/control item IDs (e.g., `OK_Item`, `OpenGL_Dialog`, `ZBuffer_Item`), OGL flags (`OGL_Flag_ZBuffer`, etc.), texture types (`OGL_Txtr_Wall`, etc.), string resource indices (`ColorPicker_PromptStrings`, etc.)
