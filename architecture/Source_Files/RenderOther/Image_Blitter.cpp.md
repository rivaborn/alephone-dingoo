# Source_Files/RenderOther/Image_Blitter.cpp

## File Purpose
Implements the `Image_Blitter` class, a wrapper around SDL surfaces that loads, manages, and renders 2D images for UI purposes in the Aleph One game engine. Provides scaling, cropping, and format-conversion utilities for cross-platform 2D rendering.

## Core Responsibilities
- Load images from multiple sources: `ImageDescriptor` objects, picture resource IDs, and raw SDL surfaces
- Manage SDL surface lifecycle (allocation, caching, deallocation) to optimize memory and rendering performance
- Handle image rescaling and maintain cached scaled surfaces to avoid recomputation during repeated renders
- Support proportional cropping region adjustments when image dimensions change
- Draw images to destination surfaces with configurable source/destination rectangles
- Query image dimensions in both original and scaled states

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SDL_Rect` | struct (SDL) | Defines rectangular regions for source/destination blitting |
| `SDL_Surface` | struct (SDL) | Represents pixel data and format for 2D images |
| `ImageDescriptor` | class (external) | Encapsulates image buffer, width, and height |
| `LoadedResource` | class (external) | Wrapper for MacOS resource handles / equivalents |

## Global / File-Static State
None.

## Key Functions / Methods

### Load(const ImageDescriptor& image)
- **Signature:** `bool Load(const ImageDescriptor& image)`
- **Purpose:** Load image data from an `ImageDescriptor` (game engine's high-level image representation)
- **Inputs:** `ImageDescriptor` with `GetBuffer()`, `GetWidth()`, `GetHeight()`
- **Outputs/Return:** `bool` ΓÇö true if load succeeded, false otherwise
- **Side effects:** Creates temporary SDL_Surface, delegates to `Load(SDL_Surface&)`, frees temporary surface
- **Calls:** `SDL_CreateRGBSurfaceFrom()`, `Load(SDL_Surface&)`, `SDL_FreeSurface()`
- **Notes:** Handles endianness via `ALEPHONE_LITTLE_ENDIAN` preprocessor guard; const_casts buffer pointer for SDL API

### Load(int picture_resource)
- **Signature:** `bool Load(int picture_resource)`
- **Purpose:** Load image from game engine's picture resource system by resource ID
- **Inputs:** Picture resource integer ID
- **Outputs/Return:** `bool` ΓÇö true if load succeeded
- **Side effects:** Allocates temporary surface via `picture_to_surface()`, delegates to `Load(SDL_Surface&)`
- **Calls:** `get_picture_resource_from_images()`, `picture_to_surface()`, `Load(SDL_Surface&)`, `SDL_FreeSurface()`

### Load(const SDL_Surface& s, const SDL_Rect& src)
- **Signature:** `bool Load(const SDL_Surface& s, const SDL_Rect& src)`
- **Purpose:** Core loading function ΓÇö copy pixel data from source SDL surface into managed internal surface
- **Inputs:** Source SDL surface and source rectangle (for loading subregions)
- **Outputs/Return:** `bool` ΓÇö true if successful, false if surface allocation fails
- **Side effects:** Calls `Unload()`, allocates `m_surface`, resets `m_src`, `m_scaled_src`, and `crop_rect` to original dimensions; caches display surface state
- **Calls:** `Unload()`, `SDL_CreateRGBSurface()`, `SDL_SetAlpha()`, `SDL_BlitSurface()`
- **Notes:** Preserves source surface's alpha flags during blit to ensure correct pixel transfer; endianness-dependent surface format creation

### Unload()
- **Signature:** `void Unload()`
- **Purpose:** Free all allocated SDL surfaces and reset internal state
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Frees `m_surface`, `m_disp_surface`, `m_scaled_surface`; resets width/height to 0
- **Calls:** `SDL_FreeSurface()`
- **Notes:** Called before loading new images and in destructor

### Rescale(int width, int height)
- **Signature:** `void Rescale(int width, int height)`
- **Purpose:** Change display scale and adjust crop rectangle proportionally
- **Inputs:** New width and height
- **Outputs/Return:** `void`
- **Side effects:** Updates `m_scaled_src` dimensions; scales `crop_rect` x/y/w/h proportionally
- **Calls:** None (direct integer arithmetic)
- **Notes:** Only recalculates if dimensions actually differ; maintains relative crop region during scaling

### Draw(SDL_Surface *dst_surface, SDL_Rect& dst, SDL_Rect& src)
- **Signature:** `void Draw(SDL_Surface *dst_surface, SDL_Rect& dst, SDL_Rect& src)`
- **Purpose:** Render the loaded image to a destination surface with optional cropping and scaling
- **Inputs:** Destination surface pointer, destination rect, source rect (crop region)
- **Outputs/Return:** `void`
- **Side effects:** Lazily creates `m_disp_surface` (display-format optimized) on first call; caches `m_scaled_surface` if scaling needed; performs SDL blit
- **Calls:** `Loaded()`, `SDL_DisplayFormatAlpha()`, `rescale_surface()`, `SDL_FreeSurface()`, `SDL_BlitSurface()`
- **Notes:** Early returns if not loaded or destination is null; selects source surface based on scaling state; implements lazy initialization and caching patterns for performance

## Control Flow Notes
Not part of main game loop but called by UI rendering systems. Typical lifecycle: `Load()` ΓåÆ (optional `Rescale()`) ΓåÆ multiple `Draw()` calls ΓåÆ `Unload()` on cleanup. Deferred surface creation (display-format surface only created on first `Draw()`) and cached scaling reduce overhead during repeated renders.

## External Dependencies
- `#include <SDL/SDL.h>` ΓÇö Cross-platform 2D graphics library
- `#include "Image_Blitter.h"` ΓÇö Class header
- `#include "images.h"` ΓÇö Game engine image resource system
- Functions from `images.h`: `get_picture_resource_from_images()`, `picture_to_surface()`, `rescale_surface()`
- Classes from external headers: `ImageDescriptor` (ImageLoader.h), `LoadedResource` (cseries.h)
