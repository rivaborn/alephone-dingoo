# Source_Files/CSeries/csfonts.cpp

## File Purpose
Manages font resource loading and text specification configuration for Mac OS/Carbon platforms. Provides utilities to retrieve font specifications from resource files, get/set font properties on graphics ports, and manage TextSpec structures used throughout the engine for text rendering.

## Core Responsibilities
- Load TextSpec font specifications from Mac resource files ('finf' resources)
- Query current font settings from the active graphics port
- Apply font settings (font, style, size) to the graphics port
- Maintain a default/null TextSpec state
- Bridge between resource-based font configuration and runtime graphics operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TextSpec | struct (from header) | Font configuration container with font ID, style flags, size, and font file paths |
| GrafPtr | typedef (Mac API) | Opaque pointer to a graphics port |
| Handle | typedef (Mac API) | Mac OS resource handle for memory-based resources |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| null_text_spec | TextSpec | static | Default/fallback TextSpec returned when resource loading fails |

## Key Functions / Methods

### GetNewTextSpec
- **Signature:** `void GetNewTextSpec(TextSpec *spec, short resid, short item)`
- **Purpose:** Load a TextSpec from a Mac resource file by resource ID and item index
- **Inputs:** Resource ID (`resid`), item index within the resource (`item`); output pointer (`spec`)
- **Outputs/Return:** Populates `*spec` with loaded data or null state if not found
- **Side effects:** Calls Mac resource API; may trigger resource loading from disk
- **Calls:** `GetResource()`, `LoadResource()` (Mac APIs)
- **Notes:** Validates item index is within bounds (0 to count-1); falls back to `null_text_spec` on any failure (missing resource, invalid index, unloaded resource)

### GetFont
- **Signature:** `void GetFont(TextSpec *spec)`
- **Purpose:** Capture the current font settings from the active graphics port
- **Inputs:** Output pointer (`spec`)
- **Outputs/Return:** Populates `*spec` with current port's font, style, and size
- **Side effects:** Queries graphics port state
- **Calls:** `GetPort()`, `GetPortTextFont()`, `GetPortTextFace()`, `GetPortTextSize()` (Mac APIs)
- **Notes:** Accessors used here work with Carbon API (opaque port datafields in Carbon vs. direct struct access in older Mac APIs)

### SetFont
- **Signature:** `void SetFont(TextSpec *spec)`
- **Purpose:** Apply font settings from a TextSpec to the active graphics port
- **Inputs:** TextSpec pointer (`spec`)
- **Outputs/Return:** None (void)
- **Side effects:** Modifies graphics port state; affects all subsequent text rendering
- **Calls:** `TextFont()`, `TextFace()`, `TextSize()` (Mac APIs)
- **Notes:** Assumes port is already active; font/style/size are applied in sequence

## Control Flow Notes
Utility module, not part of main game loop. Called during UI initialization and text rendering setup. `GetNewTextSpec()` likely called during resource loading phases; `GetFont()`/`SetFont()` called when rendering text elements that need font changes.

## External Dependencies
- **Mac OS APIs:** `<Carbon/Carbon.h>` (or fallback to `<Resources.h>`, `<Quickdraw.h>`)
  - Resource management: `GetResource()`, `LoadResource()`
  - Graphics port: `GetPort()`, `GetPortTextFont()`, `GetPortTextFace()`, `GetPortTextSize()`, `TextFont()`, `TextFace()`, `TextSize()`
- **Local:** `csfonts.h` (TextSpec struct definition), `cstypes.h` (int16, uint16 types)
