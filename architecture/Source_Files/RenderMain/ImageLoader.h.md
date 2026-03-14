# Source_Files/RenderMain/ImageLoader.h

## File Purpose
Provides an image loader interface for the Aleph One game engine. Defines `ImageDescriptor` class for holding loaded image data (pixels, mipmaps, metadata) and utilities for loading DDS-format images with format conversion and mipmap support.

## Core Responsibilities
- Load images from DDS files with configurable flags (mipmaps, DXTC compression, resize-to-POT)
- Store pixel data and metadata (dimensions, scales, format, mipmap count)
- Convert between image formats (RGBA8 Γåö DXTC3)
- Generate and access mipmap levels
- Apply alpha premultiplication
- Manage image resizing with optional total-byte constraints
- Provide copy-on-write semantics for image descriptors via template

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ImageDescriptor` | class | Holds loaded image: pixel buffer, dimensions, UVScale, format, mipmap count, premultiply flag |
| `copy_on_edit<T>` | template class | Copy-on-write wrapper; stores original + lazy-created copy |
| `ImageDescriptorManager` | typedef | Alias for `copy_on_edit<ImageDescriptor>` |
| `ImageFormat` enum | enum | Supported formats: RGBA8, DXTC1, DXTC3, DXTC5, Unknown |

## Global / File-Static State
None.

## Key Functions / Methods

### ImageDescriptor::LoadFromFile
- **Signature:** `bool LoadFromFile(FileSpecifier& File, int ImgMode, int flags, int actual_width = 0, int actual_height = 0, int maxSize = 0)`
- **Purpose:** Load image from disk file with optional dimension override and size limit.
- **Inputs:** File path, image mode (Colors/Opacity), load flags (resize, DXTC, mipmaps, etc.), optional actual dimensions, optional max buffer size
- **Outputs/Return:** `true` if successful; pixel data populated into `Pixels` buffer
- **Side effects:** Allocates `Pixels` array; modifies Width, Height, Size, MipMapCount, Format, PremultipliedAlpha
- **Calls:** `LoadDDSFromFile()`, `LoadMipMapFromFile()`, `SkipMipMapFromFile()`, `Resize()`, format converters

### ImageDescriptor::Resize (two overloads)
- **Signature (1):** `void Resize(int _Width, int _Height)`
- **Signature (2):** `void Resize(int _Width, int _Height, int _TotalBytes)`
- **Purpose:** Reallocate pixel buffer to new dimensions, optionally with explicit total byte count for mipmap storage.
- **Inputs:** new width, height, and optional total bytes
- **Outputs/Return:** (void)
- **Side effects:** Deallocates old `Pixels`; allocates new buffer; updates Width, Height, Size

### ImageDescriptor::GetMipMapPtr
- **Signature:** `uint32 *GetMipMapPtr(int Level)` and `const` variant
- **Purpose:** Return pointer to pixel data for a specific mipmap level.
- **Inputs:** mipmap level (0 = full resolution)
- **Outputs/Return:** Pointer to mipmap data; `nullptr` if level invalid
- **Side effects:** None

### ImageDescriptor::Minify
- **Signature:** `bool Minify()`
- **Purpose:** Generate next smaller mipmap level by downsampling current level.
- **Inputs:** None (operates on current buffer)
- **Outputs/Return:** `true` if successful; increments MipMapCount
- **Side effects:** Reallocates/extends `Pixels` buffer; modifies Size, MipMapCount

### ImageDescriptor::MakeRGBA / MakeDXTC3
- **Signature:** `bool MakeRGBA()`, `bool MakeDXTC3()`
- **Purpose:** Convert image format in-place.
- **Inputs:** None (operates on current buffer)
- **Outputs/Return:** `true` if successful
- **Side effects:** Reallocates `Pixels`; updates Format and Size

### ImageDescriptor::PremultiplyAlpha
- **Signature:** `void PremultiplyAlpha()`
- **Purpose:** Pre-multiply alpha into RGB channels (for blending optimization).
- **Inputs:** None
- **Outputs/Return:** (void)
- **Side effects:** Modifies pixel data in-place; sets `PremultipliedAlpha = true`

### copy_on_edit<T>::set / get / edit
- **set(const T* / T*):** Store original pointer; discard any existing copy.
- **get():** Return const pointer to data (copy if exists, else original).
- **edit():** Return mutable pointer, creating deep copy of original if needed (lazy copy semantics).
- **edit(T*):** Take ownership of provided copy; clear original.

### GetMipMapPtr (global utility)
- **Signature:** `uint32 *GetMipMapPtr(uint32 *pixels, int size, int level, int width, int height, int format)`
- **Purpose:** Compute mipmap pointer from flat pixel buffer without creating an ImageDescriptor.
- **Inputs:** pixel buffer, total size, level, base dimensions, format
- **Outputs/Return:** Pointer to mipmap in buffer
- **Calls:** Not directly observable from this file (utility)

## Control Flow Notes
Typical flow: `LoadFromFile()` reads DDS, optionally generates mipmaps with `Minify()`, converts format if needed (`MakeRGBA`/`MakeDXTC3`), applies `PremultiplyAlpha()`. Pixel data accessed via `GetBuffer()` or `GetMipMapPtr()`. Destructor auto-deallocates on scope exit.

The `copy_on_edit` wrapper allows efficient read-only sharing: multiple consumers can read the same image, but only one writer triggers a full copy.

## External Dependencies
- `DDS.h` ΓÇö `DDSURFACEDESC2` structure (DDS file format)
- `FileHandler.h` ΓÇö `FileSpecifier`, `OpenedFile` abstractions
- `cseries.h` ΓÇö platform portability macros, `uint32` type
- `<vector>` ΓÇö (for mipmap or internal storage, not visible in header)

## Notes
- Enum constants `ImageLoader_Colors`, `ImageLoader_Opacity`, and flag bits (`ImageLoader_ResizeToPowersOfTwo`, etc.) define loading behavior.
- Destructor safely deletes `Pixels` even if `nullptr`.
- `PremultipliedAlpha` is public to allow "find silhouette" code path to unset it.
- Private DDS-specific loaders (`LoadDDSFromFile`, `LoadMipMapFromFile`, `SkipMipMapFromFile`) hide format complexity.
