# Source_Files/RenderOther/Image_Blitter.h

## File Purpose
Declares the `Image_Blitter` class for loading and rendering 2D images/sprites in the UI layer. Handles image transformation (tinting, rotation, scaling) and blitting to SDL surfaces with flexible source/destination rectangles and cropping.

## Core Responsibilities
- Load images from multiple sources: `ImageDescriptor`, picture resource IDs, and SDL surfaces
- Manage image lifecycle (load, unload, track loaded state)
- Apply transformations: tinting (RGBA), rotation (degrees), scaling, and cropping
- Draw images to destination surfaces with source/dest rect control
- Maintain both original and scaled surface copies for rendering efficiency
- Query image dimensions (scaled and unscaled)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ImageDescriptor | struct (external) | Container for loaded image data, dimensions, and format |
| SDL_Surface | struct (external) | SDL graphics surface for pixel data and rendering |
| SDL_Rect | struct (external) | Rectangle definition for source/destination blitting regions |

## Global / File-Static State
None.

## Key Functions / Methods

### Image_Blitter (constructor)
- Signature: `Image_Blitter()`
- Purpose: Initialize a new blitter object
- Inputs: None
- Outputs/Return: New object instance
- Side effects: Allocates object; does not allocate image data
- Notes: No parameters; state is configured via Load() and public members

### Load (overload 1)
- Signature: `bool Load(const ImageDescriptor& image)`
- Purpose: Load image from an ImageDescriptor
- Inputs: ImageDescriptor reference
- Outputs/Return: `bool` success/failure
- Side effects: Allocates m_surface and related members; unloads previous image
- Calls: (implementation not visible)

### Load (overloads 2ΓÇô4)
- Signature: `bool Load(int picture_resource); bool Load(const SDL_Surface& s); bool Load(const SDL_Surface& s, const SDL_Rect& src)`
- Purpose: Load from resource ID, SDL_Surface, or SDL_Surface with source rect
- Inputs: Resource ID, surface, or surface + rect
- Outputs/Return: `bool` success/failure
- Side effects: Allocates/replaces m_surface, m_disp_surface, m_scaled_surface
- Notes: Multiple entry points allow flexible image sources

### Rescale
- Signature: `void Rescale(int width, int height)`
- Purpose: Scale the image to specified dimensions
- Inputs: Target width, height
- Outputs/Return: None
- Side effects: Updates m_scaled_surface and m_scaled_src

### Draw (overload 1)
- Signature: `virtual void Draw(SDL_Surface *dst_surface, SDL_Rect& dst)`
- Purpose: Draw image to destination surface using default crop_rect
- Inputs: Destination surface, destination rectangle
- Outputs/Return: None
- Side effects: Modifies pixels in dst_surface
- Calls: Delegates to Draw(dst_surface, dst, crop_rect)

### Draw (overload 2)
- Signature: `virtual void Draw(SDL_Surface *dst_surface, SDL_Rect& dst, SDL_Rect& src)`
- Purpose: Draw image with explicit source rectangle
- Inputs: Destination surface, destination rect, source rect
- Outputs/Return: None
- Side effects: Renders to dst_surface; respects tint_color and rotation
- Notes: Full control over source/dest regions; applies all transformations

### Unload
- Signature: `virtual void Unload()`
- Purpose: Release image resources
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates m_surface, m_disp_surface, m_scaled_surface
- Notes: Virtual; may be overridden in subclasses

### Dimension Queries
- Signature: `int Width(); int Height(); int UnscaledWidth(); int UnscaledHeight()`
- Purpose: Query current (scaled) and original (unscaled) image dimensions
- Inputs: None
- Outputs/Return: Dimension in pixels
- Notes: Width/Height reflect scaled dimensions if Rescale() was called

### Loaded
- Signature: `bool Loaded()`
- Purpose: Check if an image is currently loaded
- Inputs: None
- Outputs/Return: `bool` loaded state

## Control Flow Notes
This is a UI-layer rendering component. Typical usage flow:
1. Construct Image_Blitter
2. Call Load() with image source
3. Optionally call Rescale() for scaling
4. Set public transformation members (tint_color, rotation, crop_rect)
5. Call Draw() during frame/UI render pass
6. Call Unload() or destructor to release resources

Likely called from a UI rendering subsystem during the render phase, not from game logic.

## External Dependencies
- **SDL/SDL.h** ΓÇô Graphics library for surfaces and rectangles
- **ImageLoader.h** ΓÇô ImageDescriptor class for image data container
- **cseries.h** ΓÇô General utilities and type definitions (standard Aleph One utility header)

**External symbols used:**
- `SDL_Surface`, `SDL_Rect` ΓÇô Defined in SDL library
- `ImageDescriptor` ΓÇô Defined in ImageLoader.h
