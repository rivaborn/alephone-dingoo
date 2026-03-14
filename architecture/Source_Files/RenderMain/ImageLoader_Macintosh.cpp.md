# Source_Files/RenderMain/ImageLoader_Macintosh.cpp

## File Purpose
macOS-specific image loader that uses QuickTime to load image files from disk and convert them to the engine's native pixel format (RGBA). Supports both color and opacity/alpha channel loading, with optional resizing to powers of two and DDS format support.

## Core Responsibilities
- Load image files via QuickTime GraphicsImporter component
- Convert QuickTime ARGB pixel data to engine RGBA format
- Handle separate color and opacity channel loading
- Support power-of-two resizing with UV scale tracking
- Validate opacity images match color image dimensions
- Premultiply alpha channel if requested
- Manage QuickTime GWorld (offscreen graphics context) allocation and cleanup

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ImageDescriptor` | class (member function only) | Image data container with pixel buffer and metadata |
| `GraphicsImportComponent` | QuickTime typedef | QuickTime importer for image file formats |
| `GWorldPtr` | QuickTime typedef | Offscreen graphics world for pixel manipulation |
| `PixMapHandle` | QuickTime typedef | Handle to pixel map within GWorld |
| `Rect` | QuickTime struct | Image bounds (left, top, right, bottom) |

## Global / File-Static State
None.

## Key Functions / Methods

### ImageDescriptor::LoadFromFile
- **Signature:** `bool LoadFromFile(FileSpecifier& File, int ImgMode, int flags, int actual_height, int actual_width, int maxSize)`
- **Purpose:** Load an image file from disk, convert pixel data to engine format, and populate the ImageDescriptor with dimensions, pixel buffer, and scaling metadata.
- **Inputs:**
  - `File`: filesystem location of image file
  - `ImgMode`: `ImageLoader_Colors` or `ImageLoader_Opacity` (load color or alpha channel separately)
  - `flags`: bitmask controlling resize-to-POT, DDS support, premultiply, etc.
  - `actual_width`, `actual_height`, `maxSize`: optional size hints
- **Outputs/Return:** `true` on success; `false` if QuickTime unavailable, file unreadable, DDS load fails, size mismatch, or component errors.
- **Side effects:**
  - Allocates and populates `this->Pixels` buffer via `Resize()`
  - Sets `this->Width`, `this->Height`, `VScale`, `UScale`, `MipMapCount`, `PremultipliedAlpha`
  - Allocates/deallocates QuickTime GWorld and graphics importer
  - Calls `LoadDDSFromFile()` if DDS format and `ImageLoader_Colors` mode
  - Validates opacity image dimensions match preloaded color image
- **Calls:**
  - `machine_has_quicktime()` (defined elsewhere)
  - `LoadDDSFromFile()` (member, defined elsewhere)
  - `IsPresent()` (member)
  - `GetGraphicsImporterForFile()` (QuickTime)
  - `GraphicsImportGetBoundsRect()`, `NewGWorld()`, `GetGWorldPixMap()`, `LockPixels()`, `GraphicsImportSetGWorld()`, `GraphicsImportDraw()`, `CloseComponent()`, `UnlockPixels()`, `DisposeGWorld()` (all QuickTime)
  - `NextPowerOfTwo()` (defined elsewhere)
  - `Resize()` (member)
  - `GetPixBaseAddr()`, `GetPixelBasePtr()` (QuickTime/member)
  - `PIN()` macro (defined elsewhere, clamps int to range)
- **Notes:**
  - **Color mode:** Converts ARGB bytes (QuickTime) ΓåÆ RGBA uint32 (engine), sets alpha = 0xff (fully opaque). Pads rows if original width < padded width.
  - **Opacity mode:** Reads ARGB from existing pixel buffer, averages RGB to grayscale, stores as alpha channel (0ΓÇô255). Requires color image already loaded and dimensions matching.
  - **UV scaling:** Tracks fractional scaling ratios if image is resized to POT; used for texture coordinate mapping.
  - **Error path:** Cleans up QuickTime resources before returning false.
  - **Pixel layout:** Row bytes from GWorld may be padded; code accounts for rowBytes mask `0x7fff`.
  - **Premultiplication:** Deferred; flag is set but actual multiplication handled by `PremultiplyAlpha()` method (called elsewhere).

## Control Flow Notes
This is an **asset load** routine, not part of a frame loop. Called during level/resource initialization to populate image data into memory. The QuickTime importer is created, used once, and destroyed per call; no persistent state. The pixel data is copied into the ImageDescriptor buffer and can be uploaded to GPU after this function returns.

## External Dependencies
- **QuickTime (macOS):** `QuickTimeComponents.h`, `GraphicsImportComponent`, GWorld/PixMap APIs
- **macOS Frameworks:** `Carbon.h` (conditional include for Carbon.h or QuickTimeComponents.h alone)
- **Engine headers:**
  - `ImageLoader.h` ΓåÆ `ImageDescriptor` class definition, `ImageLoader_*` flags, DDS support
  - `shell.h` ΓåÆ `FileSpecifier` class, `machine_has_quicktime()`, screen_printf debugging
- **Defined elsewhere:**
  - `machine_has_quicktime()` (availability check)
  - `LoadDDSFromFile()` (DDS loader, member function)
  - `NextPowerOfTwo()` (utility)
  - `PIN()` macro (clamping)
  - `vassert()` macro (assertion with message)
  - `csprintf()` function (formatted string)
