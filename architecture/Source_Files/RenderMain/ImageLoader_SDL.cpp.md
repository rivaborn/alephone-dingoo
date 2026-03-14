# Source_Files/RenderMain/ImageLoader_SDL.cpp

## File Purpose
SDL-based image file loading implementation for the Aleph One game engine. Loads image files from disk, converts them to 32-bit RGBA SDL surfaces, and populates ImageDescriptor objects with pixel data for use in rendering (colors) or opacity maps.

## Core Responsibilities
- Load image files via SDL_image (IMG_Load) or fallback to SDL_LoadBMP
- Handle platform-specific DDS file loading via delegation
- Resize images to power-of-two dimensions when required
- Convert loaded images to 32-bit RGBA format with endianness handling
- Extract opacity channel from grayscale images
- Manage SDL surface allocation and deallocation
- Support color and opacity image modes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ImageDescriptor | class | Holds loaded image data (pixels, dimensions, scales, format) |
| SDL_Surface | struct (external) | SDL surface object wrapping pixel data and metadata |
| FileSpecifier | class | File path abstraction for cross-platform file access |

## Global / File-Static State
None

## Key Functions / Methods

### ImageDescriptor::LoadFromFile
- **Signature:** `bool LoadFromFile(FileSpecifier& File, int ImgMode, int flags, int actual_width, int actual_height, int maxSize)`
- **Purpose:** Load an image file and populate this ImageDescriptor with pixel data
- **Inputs:**
  - `File`: FileSpecifier pointing to the image file
  - `ImgMode`: ImageLoader_Colors (RGBA) or ImageLoader_Opacity (grayscale ΓåÆ alpha)
  - `flags`: Bitmask controlling resize behavior, DDS support, premultiplication, etc.
  - `actual_width`, `actual_height`: Original image dimensions (for UV scaling)
  - `maxSize`: Unused parameter (appears vestigial)
- **Outputs/Return:** `true` if load succeeded; `false` on failure
- **Side effects:**
  - Allocates/resizes internal Pixels buffer via `Resize()`
  - Updates VScale and UScale for UV coordinate mapping
  - Sets PremultipliedAlpha flag if requested
  - Allocates and frees temporary SDL_Surface objects
- **Calls (visible in this file):**
  - `LoadDDSFromFile()` (delegates for DDS files; defined elsewhere)
  - `IsPresent()` (checks if image already loaded)
  - `Resize(width, height)` (allocates pixel buffer)
  - `NextPowerOfTwo()` (rounds dimensions up; defined elsewhere)
  - `GetPixelBasePtr()` (returns pointer to pixel data)
  - `PIN()` (clamps value; defined elsewhere)
  - `SDL_LoadBMP()`, `IMG_Load()` (SDL library calls)
  - `SDL_CreateRGBSurface()`, `SDL_SetAlpha()`, `SDL_BlitSurface()`, `SDL_FreeSurface()` (SDL library calls)
- **Notes:**
  - Handles endianness via preprocessor (`ALEPHONE_LITTLE_ENDIAN`)
  - For ImageLoader_Opacity mode: validates that incoming opacity image has same dimensions/scale as color image already loaded
  - Opacity extraction: converts RGB to grayscale via simple average, clamps to [0,255]
  - Uses `memcpy` for Colors mode (direct pixel copy); loop-based conversion for Opacity mode
  - Disables SDL alpha blending before blit to prevent unwanted alpha premultiplication by SDL

## Control Flow Notes
This function is called during engine initialization to load textures and UI graphics. It serves the load phase of a texture lifecycle: `load ΓåÆ store in descriptor ΓåÆ bind to render context`. The code path splits on `ImgMode`: Colors loads a new image and resizes the descriptor; Opacity loads a second image to use as an alpha mask for an already-loaded color image.

## External Dependencies
- **Includes/Imports:**
  - `ImageLoader.h` ΓÇô ImageDescriptor class, image mode/flag constants
  - `FileHandler.h` ΓÇô FileSpecifier class
  - `<SDL_image.h>` (conditional `HAVE_SDL_IMAGE_H`) ΓÇô IMG_Load for JPEG, PNG, etc.
  - SDL library functions: surface creation, blitting, freeing
- **Defined elsewhere:**
  - `LoadDDSFromFile()` ΓÇô DDS-specific loader
  - `NextPowerOfTwo()` ΓÇô dimension rounding utility
  - `PIN()` ΓÇô clamping utility
  - `vassert()` ΓÇô assertion macro with formatted message
  - `temporary` ΓÇô global buffer for error messages (csprintf output)
