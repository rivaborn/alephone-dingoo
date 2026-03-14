# Source_Files/RenderOther/OGL_LoadScreen.cpp

## File Purpose
Implements an OpenGL-based loading screen that displays images with optional progress bars during engine/game loading. Part of the Aleph One game engine's rendering system, managing screen display and cleanup during initialization phases.

## Core Responsibilities
- Singleton pattern management for global loading screen access
- Image loading from files and GPU texture preparation
- Screen-space rendering with aspect-ratio-aware scaling/stretching
- Progress bar overlay with color customization
- OpenGL state management (matrix transforms, texture binding, vertex drawing)
- Resource cleanup and screen buffer swaps

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_LoadScreen` | class | Singleton managing load screen rendering and state |
| `ImageDescriptor` | struct (external) | Stores loaded image data and metadata |
| `OGL_Blitter` | class (external) | Handles GPU rendering of 2D images |
| `SDL_Rect` | struct (external) | Destination rectangle for scaled image placement |
| `rgb_color` | struct (external) | 16-bit color (red/green/blue) for progress bar |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `OGL_LoadScreen::instance_` | `OGL_LoadScreen*` | static (file-scoped via class) | Singleton instance pointer |

## Key Functions / Methods

### `instance()`
- **Signature:** `static OGL_LoadScreen *instance()`
- **Purpose:** Lazy-construct and return singleton instance
- **Inputs:** None
- **Outputs/Return:** Pointer to static `OGL_LoadScreen` instance
- **Side effects:** First call allocates `instance_` via `new`
- **Calls:** None visible in this file
- **Notes:** Classic singleton pattern; no thread safety

### `Start()`
- **Signature:** `bool Start()`
- **Purpose:** Load image file, configure rendering geometry, initialize progress display
- **Inputs:** Reads `path`, `stretch`, `x`, `y`, `w`, `h` from member state (set via `Set()`)
- **Outputs/Return:** `true` if successful; sets `use = true/false` and returns it
- **Side effects:** Loads GPU texture via `blitter.Load(image)`; calls `OGL_ClearScreen()`; modifies `m_dst`, `x_offset`, `y_offset`, `x_scale`, `y_scale`
- **Calls:** `File.SetNameWithPath()`, `image.LoadFromFile()`, `blitter.Load()`, `SDL_GetVideoSurface()`, `bound_screen()`, `OGL_ClearScreen()`, `Progress(0)`
- **Notes:** Scales image to fit screen while preserving aspect ratio (unless `stretch=true`); centers in viewport; early return on file/load failure sets `use=false`

### `Stop()`
- **Signature:** `void Stop()`
- **Purpose:** Finalize loading screen and clean up
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears screen, swaps buffers, calls `Clear()`
- **Calls:** `OGL_ClearScreen()`, `OGL_SwapBuffers()`, `Clear()`
- **Notes:** Simple wrapper sequence; typically called after loading completes

### `Progress(int progress)`
- **Signature:** `void Progress(const int progress)`
- **Purpose:** Render current frame: image + optional progress bar overlay
- **Inputs:** `progress` ΓÇö percentage value (0ΓÇô100)
- **Outputs/Return:** None (screen output only)
- **Side effects:** Clears screen, modifies GL matrix stack (push/transform/pop), swaps buffers
- **Calls:** `OGL_ClearScreen()`, `OGL_Blitter::BoundScreen()`, `blitter.Draw()`, `glMatrixMode()`, `glPushMatrix()`, `glTranslated()`, `glScaled()`, `glDisable()`, `SglColor3us()`, `glBegin()`, `glVertex3f()`, `glEnd()`, `glEnable()`, `glPopMatrix()`, `OGL_SwapBuffers()`
- **Notes:** Progress bar orientation is vertical or horizontal depending on `h > w`; uses `colors[0]` for background, `colors[1]` for fill; GL matrix stack properly balanced

### `Set(const vector<char> Path, bool Stretch)` (1st overload)
- **Signature:** `void Set(const vector<char> Path, bool Stretch)`
- **Purpose:** Configure load screen with image path and scaling mode; disable progress bar
- **Inputs:** `Path` ΓÇö file path; `Stretch` ΓÇö full-screen stretch flag
- **Outputs/Return:** None
- **Side effects:** Sets member variables; calls second `Set()` overload
- **Calls:** `Set(Path, Stretch, 0, 0, 0, 0)`; sets `useProgress = false`
- **Notes:** Convenience wrapper disabling progress bar

### `Set(const vector<char> Path, bool Stretch, short X, short Y, short W, short H)` (2nd overload)
- **Signature:** `void Set(const vector<char> Path, bool Stretch, short X, short Y, short W, short H)`
- **Purpose:** Full configuration of image path, scaling, and progress bar geometry
- **Inputs:** `Path` ΓÇö file path; `Stretch` ΓÇö scaling mode; `X, Y, W, H` ΓÇö progress bar bounds in image-local coords
- **Outputs/Return:** None
- **Side effects:** Stores all parameters in members; resets `percent=0`; sets `use=true, useProgress=true`
- **Calls:** None
- **Notes:** Called by both public `Set()` overloads; progress bar coords are relative to scaled image, not screen

### `Clear()`
- **Signature:** `void Clear()`
- **Purpose:** Reset all state, deallocate resources, disable loading screen
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears screen 3 times, swaps buffers; sets `use=false, useProgress=false`; clears `path`, `image`; unloads blitter
- **Calls:** `OGL_ClearScreen()` (3x), `OGL_SwapBuffers()`, `image.Clear()`, `blitter.Unload()`
- **Notes:** Triple-clear pattern may ensure buffer visibility on some platforms

## Control Flow Notes
**Initialization phase:**
1. `Set()` ΓÇö configure (no rendering)
2. `Start()` ΓÇö load image, prepare GPU, call `Progress(0)`
3. Loop: `Progress(percent)` ΓÇö render each frame during loading
4. `Stop()` ΓåÆ `Clear()` ΓÇö cleanup after loading complete

This file is part of the engine's **loading/startup sequence**, called before main game rendering begins. It does not participate in the main frame update/render loop.

## External Dependencies
- **Includes (this file):**
  - `OGL_LoadScreen.h` ΓÇö class definition
  - `screen.h` ΓÇö screen surface API
  - `OGL_Setup.h` ΓÇö `SglColor3us()` macro/function

- **External symbols used:**
  - `OGL_SwapBuffers()`, `OGL_ClearScreen()`, `bound_screen()` ΓÇö defined elsewhere, render API
  - `SDL_GetVideoSurface()` ΓÇö SDL 1.x video API
  - OpenGL: `glMatrixMode()`, `glPushMatrix()`, `glTranslated()`, `glScaled()`, `glDisable()`, `glEnable()`, `glBegin()`, `glVertex3f()`, `glEnd()`, `GL_MODELVIEW`, `GL_TEXTURE_2D`, `GL_QUADS`
  - `FileSpecifier`, `ImageDescriptor`, `ImageLoader_Colors` ΓÇö image loading framework
  - `OGL_Blitter` ΓÇö 2D rendering utility
