# Source_Files/CSeries/cscluts.h

## File Purpose
Defines color lookup table (CLUT) structures and initialization functions for the Aleph One game engine. Provides abstractions for managing color palettes on both Macintosh and cross-platform systems, supporting 256-color indexed graphics.

## Core Responsibilities
- Define color data structures (`rgb_color`, `color_table`) for palette representation
- Declare color table builder functions for both Mac-specific and generic implementations
- Define system color palette enum indices (highlight colors, grayscale tones)
- Declare global color constants (`rgb_black`, `rgb_white`, `system_colors[]`)
- Provide Mac Carbon bridge for native color table construction

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` | struct | 16-bit RGB color component storage (red, green, blue) |
| `color_table` | struct | Indexed color palette container; holds up to 256 colors with count |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rgb_black` | `RGBColor` | global | Predefined black color constant |
| `rgb_white` | `RGBColor` | global | Predefined white color constant |
| `system_colors[]` | `RGBColor[NUM_SYSTEM_COLORS]` | global | System palette for UI (gray15Percent, windowHighlight, etc.) |

## Key Functions / Methods

### build_macintosh_color_table
- **Signature:** `CTabHandle build_macintosh_color_table(color_table *table)`
- **Purpose:** Convert engine color_table struct to native Macintosh ColorTable handle (Mac-only).
- **Inputs:** Pointer to a `color_table` struct.
- **Outputs/Return:** `CTabHandle` (Mac Carbon opaque handle to color table).
- **Side effects:** Allocates Mac-native resource memory.
- **Calls:** Not defined in this file; called from platform-specific implementation.
- **Notes:** Guarded by `#ifdef mac`; abstraction layer for Mac Graphics API.

### build_color_table
- **Signature:** `void build_color_table(color_table *table, LoadedResource &clut)`
- **Purpose:** Populate a `color_table` struct from a loaded resource (cross-platform).
- **Inputs:** Pointer to `color_table` (destination) and `LoadedResource` reference (source CLUT data).
- **Outputs/Return:** None (modifies `table` in-place).
- **Side effects:** Reads from resource object; writes to color_table structure.
- **Calls:** Not defined in this file; implemented elsewhere.
- **Notes:** Generic loader; abstracts resource format parsing from color table representation.

## Control Flow Notes
Likely called during engine initialization (render system setup) to load graphical resources and populate the renderer's palette. The system colors are initialized before frame rendering begins. On Mac, native CLUT conversion bridges between engine representation and OS graphics libraries.

## External Dependencies
- **Includes:** `cstypes.h` (integer types: `uint16`, `short`)
- **Forward declared:** `LoadedResource` class (resource management, defined elsewhere)
- **External symbols used:**
  - `CTabHandle` (Mac Carbon type, defined in Carbon headers)
  - `RGBColor` (Mac Carbon struct for color, defined in Carbon headers)
  - `NUM_SYSTEM_COLORS` (enum constant, `= 2` per enum definition)
