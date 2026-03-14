# Source_Files/RenderMain/ImageLoader_Shared.cpp

## File Purpose
Implements image descriptor operations and DDS (DirectDraw Surface) file loading with support for DXTC texture compression formats. Provides mipmap chain management, format conversions (DXTCΓåöRGBA), DXTC decompression, and alpha premultiplication.

## Core Responsibilities
- **Mipmap management**: calculate sizes, retrieve pointers, traverse mipmap chains
- **DDS file parsing**: parse headers, validate formats, load/skip mipmap levels
- **Format handling**: detect and convert RGBA8, DXTC1, DXTC3, DXTC5 formats
- **DXTC decompression**: decompress all three DXTC variants to uncompressed RGBA
- **Buffer resizing**: reallocate pixel storage and manage memory
- **Image scaling**: downscale via mipmaps or GLU when needed
- **Alpha preprocessing**: premultiply alpha channel before upload

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Color8888` | struct | 32-bit RGBA color (r, g, b, a bytes) |
| `Color565` | struct | 16-bit RGB color with 5:6:5 bit layout; used in DXTC color blocks |
| `DXTColBlock` | struct | DXTC color reference and 2-bit color index rows |
| `DXTAlphaBlockExplicit` | struct | DXTC3 explicit 4-bit per-pixel alpha data |
| `DXTAlphaBlock3BitLinear` | struct | DXTC5 two-endpoint interpolated alpha data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `padfour()` | inline function | file-static | Pad integer to next multiple of 4 (alignment) |
| `log2()` | inline function | file-static | Compute logΓéé as std::log(x)/std::log(2) |
| `DecompressDXTC1/3/5()` | static functions | file-static | Forward declarations; implementations decompile DXTC blocks |

## Key Functions / Methods

### ImageDescriptor::GetMipMapSize
- **Signature:** `int GetMipMapSize(int level) const`
- **Purpose:** Calculate byte size of a single mipmap level (level 0 is full resolution).
- **Inputs:** `level` (0 = base, 1 = half-width/height, etc.)
- **Outputs/Return:** Size in bytes; 0 if level is invalid.
- **Side effects:** none
- **Calls:** none
- **Notes:** Divides dimensions by 2Γü┐; accounts for format-specific packing (DXTC uses 4├ù4 blocks).

### ImageDescriptor::GetMipMapPtr (const and non-const versions)
- **Signature:** `const uint32 *GetMipMapPtr(int Level) const` / `uint32 *GetMipMapPtr(int Level)`
- **Purpose:** Return pointer to mipmap level data within the pixel buffer.
- **Inputs:** `Level` (0 = base)
- **Outputs/Return:** Pointer to level's pixel data, or NULL if out of bounds.
- **Side effects:** none
- **Calls:** `GetMipMapSize()`, `GetBufferSize()`, `GetBuffer()`
- **Notes:** Accumulates sizes of all prior levels to compute offset; no bounds checking beyond buffer size.

### ImageDescriptor::Resize
- **Signature:** `void Resize(int _Width, int _Height)` / `void Resize(int _Width, int _Height, int _TotalBytes)`
- **Purpose:** Deallocate old pixel buffer and allocate new one with specified dimensions.
- **Inputs:** width, height, optional explicit total byte count (default: 4 bytes per pixel).
- **Outputs/Return:** none
- **Side effects:** `delete[] Pixels`; new allocation; updates `Width`, `Height`, `Size`.
- **Calls:** `new`, `delete[]`
- **Notes:** Overload without `_TotalBytes` assumes RGBA8 (4 bpp). No check for allocation failure.

### ImageDescriptor::Minify
- **Signature:** `bool Minify()`
- **Purpose:** Reduce image size by half, either from existing mipmap or by scaling.
- **Inputs:** none (uses this image's state)
- **Outputs/Return:** true if successful; false if image is 1├ù1 or non-RGBA unscalable.
- **Side effects:** Reallocates `Pixels`; updates `Width`, `Height`, `Size`, `MipMapCount`.
- **Calls:** `GetMipMapSize()`, `GetMipMapPtr()`, `gluScaleImage()` (if OpenGL active), `memcpy()`, `delete[]`, `new`
- **Notes:** Prefers mipmap chain; falls back to GLU scaling if available. Fails if GL is required but inactive.

### ImageDescriptor::LoadDDSFromFile
- **Signature:** `bool LoadDDSFromFile(FileSpecifier& File, int flags, int actual_width = 0, int actual_height = 0, int maxSize = 0)`
- **Purpose:** Parse DDS file header and load texture data, with optional format conversion and mipmap management.
- **Inputs:** `File` (file path), `flags` (ImageLoader_* options), `actual_width`/`actual_height` (override dimensions for scaling), `maxSize` (skip large mipmaps).
- **Outputs/Return:** true if successful; false if header invalid, unsupported format, or I/O error.
- **Side effects:** Reads file; calls `Resize()`; allocates `Pixels`; updates `Format`, `Width`, `Height`, `MipMapCount`, `VScale`, `UScale`.
- **Calls:** `AIStreamLE` (little-endian stream parsing), `LoadMipMapFromFile()`, `SkipMipMapFromFile()`, `Minify()`, `MakeRGBA()`, `MakeDXTC3()`.
- **Notes:** 
  - Validates DDS magic and header size.
  - Rejects cubemaps and volume textures.
  - Computes mipmap count from dimensions if present in header; validates completeness.
  - Resizes to next power of 2 if flag set; computes UScale/VScale for texture coordinate adjustment.
  - Handles DXTC1, DXTC3, DXTC5, and RGBA8 formats.
  - Converts missing higher mipmaps by copying lower ones.

### ImageDescriptor::LoadMipMapFromFile
- **Signature:** `bool LoadMipMapFromFile(OpenedFile& file, int flags, int level, DDSURFACEDESC2 &ddsd, int skip)`
- **Purpose:** Read a single mipmap level from DDS file into pre-allocated buffer.
- **Inputs:** `file` (opened DDS), `flags`, `level` (0=base), `ddsd` (DDS header), `skip` (offset to apply).
- **Outputs/Return:** true on success; false on I/O error or buffer overflow.
- **Side effects:** Reads from file; writes to `GetBuffer()`.
- **Calls:** `GetBuffer()`, `GetMipMapSize()`, `GetBufferSize()`, `file.Read()`, SDL surface functions (`SDL_CreateRGBSurfaceFrom`, `SDL_BlitSurface`, `SDL_FreeSurface`), `memset()`, `memcpy()`.
- **Notes:** 
  - RGBA8: uses SDL to convert source format (24/32-bit) to RGBA8.
  - DXTC1/3/5: directly copies compressed blocks from file; pads dimensions to 4├ù4 boundaries.

### ImageDescriptor::SkipMipMapFromFile
- **Signature:** `bool SkipMipMapFromFile(OpenedFile& File, int flags, int level, DDSURFACEDESC2 &ddsd)`
- **Purpose:** Advance file position past a mipmap level without loading.
- **Inputs:** `File`, `flags`, `level`, `ddsd` (header).
- **Outputs/Return:** true if successful; false on I/O error.
- **Side effects:** Updates file position.
- **Calls:** `File.GetPosition()`, `File.SetPosition()`.
- **Notes:** Computes byte size based on format and calls seek; used when maxSize skips leading mipmaps.

### ImageDescriptor::MakeRGBA
- **Signature:** `bool MakeRGBA()`
- **Purpose:** Decompress DXTC format to uncompressed RGBA8 (all mipmaps).
- **Inputs:** none (uses current pixel data and format)
- **Outputs/Return:** true on success; false if not DXTC1/3/5.
- **Side effects:** Allocates new `Pixels` buffer; updates `Format` to RGBA8; deallocates old buffer.
- **Calls:** `DecompressDXTC1/3/5()`, `new`, `delete[]`.
- **Notes:** Processes each mipmap level independently; copies pointer and clears temporary descriptor.

### ImageDescriptor::MakeDXTC3
- **Signature:** `bool MakeDXTC3()`
- **Purpose:** Convert DXTC1 to DXTC3 by adding explicit alpha blocks.
- **Inputs:** none (uses current DXTC1 data)
- **Outputs/Return:** true if format is DXTC1; false otherwise.
- **Side effects:** Doubles pixel buffer size; updates `Format`, `Size`; deallocates old buffer.
- **Calls:** `new`, `memset()`, `memcpy()`, `delete[]`.
- **Notes:** Fills alpha blocks (8 bytes per block) with 0xFF (fully opaque); color data unchanged.

### ImageDescriptor::PremultiplyAlpha
- **Signature:** `void PremultiplyAlpha()`
- **Purpose:** Multiply RGB components by alpha, in-place, for blending without alpha correction.
- **Inputs:** none (modifies RGBA8 pixels in-place)
- **Outputs/Return:** none
- **Side effects:** Updates `Pixels` (per-pixel); sets `PremultipliedAlpha = true`.
- **Calls:** (intrinsic bit/byte operations)
- **Notes:** 
  - Skips fully opaque (alpha=0xFF) and fully transparent (alpha=0x00) pixels.
  - Uses byte-level accessors for endianness handling; reads alpha from byte 3 on little-endian.

### DecompressDXTC1
- **Signature:** `static bool DecompressDXTC1(uint32 *out, int width, int height, uint32 *in)`
- **Purpose:** Decompress DXTC1 (4-bit color index per pixel) to RGBA.
- **Inputs:** `out` (output RGBA buffer), `width`/`height`, `in` (DXTC1 data).
- **Outputs/Return:** true (always succeeds).
- **Side effects:** Writes to `out`.
- **Calls:** `SDL_SwapLE16/32()`.
- **Notes:** 
  - Processes 4├ù4 blocks; each block references two 16-bit color endpoints and a 2-bit index bitmask.
  - Derives 4 colors: two from endpoints, two interpolated (or two + transparent).
  - Uses bit-shifting to extract indices; skips pixels outside image bounds.

### DecompressDXTC3
- **Signature:** `static bool DecompressDXTC3(uint32 *out, int width, int height, uint32 *in)`
- **Purpose:** Decompress DXTC3 (DXTC1 colors + explicit 4-bit alpha) to RGBA.
- **Inputs:** `out`, `width`/`height`, `in`.
- **Outputs/Return:** true (always succeeds).
- **Side effects:** Writes to `out`.
- **Calls:** `SDL_SwapLE16/32()`.
- **Notes:** 
  - Identical to DXTC1 color path, plus explicit alpha.
  - 8 bytes of 4-bit alpha (duplicated to 8-bit) per block; applied separately from color indices.

### DecompressDXTC5
- **Signature:** `static bool DecompressDXTC5(uint32 *out, int width, int height, uint32 *in)`
- **Purpose:** Decompress DXTC5 (DXTC1 colors + linear-interpolated 3-bit alpha) to RGBA.
- **Inputs:** `out`, `width`/`height`, `in`.
- **Outputs/Return:** true (always succeeds).
- **Side effects:** Writes to `out`.
- **Calls:** `SDL_SwapLE32()`.
- **Notes:** 
  - Colors: identical to DXTC1.
  - Alpha: two endpoint bytes interpolate 6 or 8 intermediate values; 3-bit indices select from palette.
  - Processes alpha in two passes (3 bytes each) to handle 6-byte layout.

## Control Flow Notes
- **Initialization**: `ImageDescriptor` constructor; destructor deallocates `Pixels`.
- **Load time**: `LoadDDSFromFile()` ΓåÆ parse header ΓåÆ `LoadMipMapFromFile()` or `SkipMipMapFromFile()` ΓåÆ optional `MakeRGBA()` / `MakeDXTC3()`.
- **Runtime**: `GetMipMapPtr()` / `GetMipMapSize()` for GPU upload; `Minify()` for fallback scaling.
- **Preprocessing**: `PremultiplyAlpha()` before GPU bind.

## External Dependencies
- **AStream.h**: `AIStreamLE` for little-endian binary parsing.
- **DDS.h**: `DDSURFACEDESC2`, `DDSD_*`, `DDPF_*`, `DDSCAPS_*` flag constants.
- **ImageLoader.h**: `ImageDescriptor` class definition.
- **SDL.h**, **SDL_endian.h**: `SDL_CreateRGBSurfaceFrom()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`, `SDL_SwapLE16/32()` for surface conversion and endian swapping.
- **OpenGL** (conditional `HAVE_OPENGL`): `gluScaleImage()`, `OGL_IsActive()` for image downscaling.
- **cstypes.h**: `uint32`, `uint16`, `uint8` type definitions; `FOUR_CHARS_TO_INT()` macro.
- **stdlib.h**, **cmath**: `std::log()` for log2 computation.
