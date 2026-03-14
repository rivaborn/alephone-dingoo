# Source_Files/RenderOther/Shape_Blitter.h

## File Purpose
Provides a utility class for rendering 2D bitmap images from a Shapes resource file (Marathon-format graphics) to SDL surfaces or OpenGL targets. Handles scaling, tinting, rotation, and cropping of shape bitmaps for UI and texture drawing.

## Core Responsibilities
- Load and manage shape descriptors from the game's resource collections
- Scale shapes to fit arbitrary destination dimensions on demand
- Render shapes via both SDL (CPU) and OpenGL (GPU) paths
- Apply visual effects: color tinting, rotation about center, and rectangular cropping
- Manage SDL surface memory and lifecycle for scaled versions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef | Identifies a shape by collection and shape index (from map.h) |
| `SDL_Rect` | struct | Rectangle region for source/destination blitting |
| `SDL_Surface` | struct | SDL pixel buffer for bitmap data |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor
- Signature: `Shape_Blitter(short collection, short texture_index, short texture_type, short clut_index = 0)`
- Purpose: Initialize blitter with a shape descriptor and texture classification
- Inputs: collection index, texture index, texture type (wall/landscape/sprite/weapon/interface), optional CLUT (color lookup table) index
- Outputs/Return: Constructed Shape_Blitter instance
- Side effects: Loads shape data; allocates m_surface and m_src from game resources
- Notes: texture_type determines how the shape is interpreted (5 types defined as enums at top of file)

### Rescale
- Signature: `void Rescale(int width, int height)`
- Purpose: Rescale the shape bitmap to fit the given dimensions
- Inputs: target width and height in pixels
- Outputs/Return: (none; modifies internal state)
- Side effects: Creates/reallocates m_scaled_surface and updates m_scaled_src
- Calls: Likely SDL surface creation/copying routines
- Notes: Called before rendering to match destination rectangle size

### Width / Height
- Signature: `int Width()`, `int Height()`
- Purpose: Return current scaled dimensions
- Outputs/Return: Pixel dimensions of scaled surface (after Rescale)

### UnscaledWidth / UnscaledHeight
- Signature: `int UnscaledWidth()`, `int UnscaledHeight()`
- Purpose: Return original unscaled shape dimensions
- Outputs/Return: Original pixel dimensions from shape resource

### OGL_Draw
- Signature: `void OGL_Draw(SDL_Rect& dst)`
- Purpose: Render the shape using OpenGL
- Inputs: Destination rectangle (screen coordinates)
- Side effects: GPU state changes; texture upload/draw calls
- Notes: High-performance rendering path; respects tint_color, rotation, crop_rect

### SDL_Draw
- Signature: `void SDL_Draw(SDL_Surface *dst_surface, SDL_Rect& dst)`
- Purpose: Render the shape using SDL blitting
- Inputs: Target SDL surface, destination rectangle
- Side effects: Pixel writes to dst_surface
- Notes: Software (CPU) rendering path; may be slower but more compatible

### Destructor
- Signature: `~Shape_Blitter()`
- Purpose: Release allocated SDL surfaces
- Side effects: Frees m_surface and m_scaled_surface

## Control Flow Notes
This is a 2D rendering utility used during the UI/render phase, independent of the game's main physics/update loop. It bridges the shape resource system (from map.h) and SDL/OpenGL rendering targets. The dual rendering paths allow UI to work with different graphics backends.

## External Dependencies
- **cseries.h**: Base engine types and utilities
- **map.h**: shape_descriptor type; shape resource system
- **SDL/SDL.h**: SDL_Rect, SDL_Surface, rendering primitives
- **\<vector\>, \<set\>**: STL containers (included but usage not visible in header)
- **shape_descriptors.h**: Shape descriptor type definitions (transitively via map.h)
