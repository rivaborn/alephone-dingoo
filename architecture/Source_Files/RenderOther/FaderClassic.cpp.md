# Source_Files/RenderOther/FaderClassic.cpp

## File Purpose
Implements direct video device gamma table manipulation to animate screen color lookup tables on macOS Classic. Part of a color fading system that works in Carbon environments by bypassing high-level APIs and directly accessing the graphics device driver.

## Core Responsibilities
- Exports `animate_screen_clut_classic()` function to apply new color tables via gamma manipulation
- Retrieves current gamma table from video device driver using Control status calls
- Converts 16-bit packed RGB color data to planar format (all reds, then greens, then blues)
- Handles both high bit-depth (>8 bits) and low bit-depth (Γëñ8 bits) gamma table entries
- Validates device state (Control driver loaded, color capable, valid gamma table)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` | struct | Holds 16-bit red, green, blue components |
| `color_table` | struct | Container for up to 256 RGB colors with count |
| `CntrlParam` | struct (MacOS) | Control device driver parameter block |
| `VDGammaRecord` | struct (MacOS) | Video device gamma table metadata and pointer |
| `GammaTblPtr` | typedef (MacOS) | Pointer to gamma table structure |

## Global / File-Static State
None.

## Key Functions / Methods

### animate_screen_clut_classic
- **Signature:** `void animate_screen_clut_classic(struct color_table *color_table, GDHandle DevHdl)`
- **Purpose:** Apply a new color table to the display by updating the video device's gamma table in place.
- **Inputs:** 
  - `color_table`: pointer to source RGB palette (Γëñ256 entries)
  - `DevHdl`: graphics device handle (device driver reference)
- **Outputs/Return:** void
- **Side effects:** 
  - Reads gamma table from device driver (PBStatus call)
  - Modifies gamma table in device memory
  - Writes updated gamma table back to device driver (Control call)
  - Direct hardware state change
- **Calls:** 
  - `PBStatus()` with `cscGetGamma` code to fetch current gamma table
  - `Control()` with `cscSetGamma` code to apply new gamma table
- **Notes:** 
  - Returns early (no-op) if Control driver not loaded, if PBStatus fails, if device is grayscale (<3 color channels), or if gamma table pointer is null
  - Assumes planar layout in gamma table (red entries, then green, then blue)
  - Right-shifts color values from 16-bit down to actual gamma entry bit depth to avoid overflow
  - Iterates once per color channel, writing to offset positions in gamma table

## Control Flow Notes
Part of a screen fading system; called when palette animations or color transitions need hardware-level updates in macOS Classic. Likely invoked from a higher-level fader manager during frame updates or transition sequences. Platform-specific to old macOS; no analogue in modern systems.

## External Dependencies
- **macOS Carbon/Classic APIs:** Control, PBStatus, GDHandle device structures, Devices.h, Quickdraw.h, Video.h
- **Utility macros:** MIN, MAX (from csmacros.h)
- **Platform:** macOS Classic only; requires working Control device driver and QuickDraw graphics subsystem
