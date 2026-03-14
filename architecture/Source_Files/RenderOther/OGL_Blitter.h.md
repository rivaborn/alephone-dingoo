# Source_Files/RenderOther/OGL_Blitter.h

## File Purpose
Defines `OGL_Blitter`, an OpenGL-based image blitter for rendering SDL surfaces and ImageDescriptor objects to screen. Inherits from `Image_Blitter` and manages OpenGL texture tiling, loading, and rendering operations.

## Core Responsibilities
- Manage OpenGL texture creation, binding, and deletion
- Tile large images into fixed-size (256├ù256) textures for GPU memory efficiency
- Provide overloaded `Draw()` methods for rendering to destination rectangles
- Maintain a global registry of all active `OGL_Blitter` instances
- Expose static utility methods for screen resolution and global texture management
- Handle platform-specific OpenGL header inclusion (macOS/Linux/Windows)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OGL_Blitter | class | OpenGL-accelerated image renderer; inherits Image_Blitter |
| m_blitter_registry | static set<OGL_Blitter*> | Global registry tracking all active blitter instances |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| m_blitter_registry | set<OGL_Blitter*> | static | Registry of all currently active OGL_Blitter instances; used by `StopTextures()` to unload all textures |
| tile_size | const int (256) | static | Fixed tile width/height for texture subdivision |

## Key Methods

### OGL_Blitter (Constructor)
- **Signature:** `OGL_Blitter()`
- **Purpose:** Initialize a new blitter instance
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Likely registers `this` in `m_blitter_registry`
- **Calls:** Not inferable from header
- **Notes:** Precedes texture loading

### Unload
- **Signature:** `void Unload()`
- **Purpose:** Release GPU resources (textures and metadata); virtual override of Image_Blitter::Unload
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Unloads OpenGL textures; clears `m_rects` and `m_refs`; sets `m_textures_loaded = false`
- **Calls:** `_UnloadTextures()` (inferred)
- **Notes:** Called on image reload or cleanup

### Draw (Overloads)
- **Signature:** 
  - `void Draw(SDL_Surface *dst_surface, SDL_Rect& dst, SDL_Rect& src)`
  - `void Draw(const SDL_Rect& dst)`  
  - `void Draw(const SDL_Rect& dst, const SDL_Rect& src)`
- **Purpose:** Render the loaded image to screen at destination and source rectangles
- **Inputs:** `dst_surface` (SDL surface), `dst`/`src` (rectangles defining destination and source regions)
- **Outputs/Return:** None
- **Side effects:** GPU state changes; texture binding
- **Calls:** `_Draw()` (private implementation)
- **Notes:** Overload without `dst_surface` uses inherited behavior; overload with only `dst` uses `crop_rect` as source

### StopTextures (Static)
- **Signature:** `static void StopTextures()`
- **Purpose:** Unbind and release all GPU textures for all active blitters
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Unload()` on every blitter in `m_blitter_registry`
- **Calls:** Not inferable from header
- **Notes:** Likely called during mode switches or shutdown

### ScreenWidth / ScreenHeight / BoundScreen (Static)
- **Signature:** 
  - `static int ScreenWidth()`
  - `static int ScreenHeight()`
  - `static void BoundScreen()`
- **Purpose:** Query current screen resolution and bind screen framebuffer for drawing
- **Inputs:** None
- **Outputs/Return:** int (width/height); void (bind)
- **Side effects:** `BoundScreen()` likely sets OpenGL viewport/projection
- **Calls:** Not inferable from header
- **Notes:** Utility methods for renderer coordination

### ~OGL_Blitter (Destructor)
- **Signature:** `~OGL_Blitter()`
- **Purpose:** Clean up and deregister on destruction
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Unload()`; removes `this` from `m_blitter_registry`
- **Calls:** `Unload()`
- **Notes:** Ensures safe cleanup of GPU resources

## Control Flow Notes
- **Load phase:** Constructor registers blitter; inherited `Load()` methods populate image data
- **Render phase:** `_LoadTextures()` called on first `Draw()` (lazy loading); `_Draw()` iterates tile rectangles and renders each
- **Unload/Shutdown phase:** `Unload()` or `StopTextures()` releases GPU memory; destructor deregisters from global set

## External Dependencies
- **Headers:** `cseries.h` (core types), `ImageLoader.h` (ImageDescriptor), `Image_Blitter.h` (base class)
- **SDL:** `SDL_Surface`, `SDL_Rect`, `SDL.h`
- **OpenGL:** `gl.h`, `glu.h`, `glext.h` (platform-conditional includes; HAVE_OPENGL guard)
- **STL:** `<vector>` (m_rects, m_refs), `<set>` (m_blitter_registry)
- **Symbols from elsewhere:** `Image_Blitter` base class (Load, crop_rect, m_surface inheritance)
