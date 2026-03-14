# Source_Files/RenderOther/OGL_LoadScreen.h

## File Purpose
Defines a singleton OpenGL-based load screen manager that displays images with optional progress indicators during loading phases. Handles image rendering, scaling, positioning, and color palette management for loading UIs.

## Core Responsibilities
- Singleton instance lifecycle (Start/Stop)
- Load and display images (full-screen or custom-positioned)
- Update and render progress percentage
- Manage color palette for progress indicator
- Handle image stretching and offset scaling
- Manage OpenGL texture resources

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_LoadScreen` | class | Singleton managing OpenGL load screen rendering |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `instance_` | `OGL_LoadScreen*` | static class member | Singleton instance pointer |

## Key Functions / Methods

### `instance()`
- Signature: `static OGL_LoadScreen *instance()`
- Purpose: Return singleton instance (lazy initialization pattern implied)
- Outputs/Return: Pointer to OGL_LoadScreen singleton
- Notes: Standard singleton accessor

### `Start()`
- Signature: `bool Start()`
- Purpose: Initialize and begin rendering the load screen
- Outputs/Return: `bool` (success indicator)
- Side effects: Allocates/loads textures, sets up rendering state

### `Stop()`
- Signature: `void Stop()`
- Purpose: Clean up and stop rendering load screen
- Side effects: Deallocates textures, releases OpenGL resources

### `Progress(const int percent)`
- Signature: `void Progress(const int percent)`
- Purpose: Update progress percentage for progress bar rendering
- Inputs: `percent` ΓÇö progress value (0ΓÇô100 implied)
- Side effects: Updates `percent` member, triggers re-render

### `Set()` (overload 1)
- Signature: `void Set(const vector<char> Path, bool Stretch)`
- Purpose: Load and display image full-screen, optionally stretched
- Inputs: `Path` ΓÇö file path; `Stretch` ΓÇö enable scaling to fit screen

### `Set()` (overload 2)
- Signature: `void Set(const vector<char> Path, bool Stretch, short X, short Y, short W, short H)`
- Purpose: Load image with custom position and dimensions
- Inputs: `Path` ΓÇö file path; `Stretch` ΓÇö scaling mode; `X`, `Y` ΓÇö offset; `W`, `H` ΓÇö size

### `Clear()`
- Signature: `void Clear()`
- Purpose: Reset load screen state, clear loaded image
- Side effects: Clears `path`, `image`, resets flags

### `Use()`
- Signature: `bool Use()`
- Purpose: Query whether load screen is currently active
- Outputs/Return: `use` flag value

### `Colors()`
- Signature: `rgb_color *Colors()`
- Purpose: Access color palette for progress bar (e.g., foreground/background)
- Outputs/Return: Pointer to 2-element `rgb_color` array

## Control Flow Notes
Used during initialization or asset-loading phases: caller invokes `Start()`, sets image via `Set()`, updates progress via `Progress()` in a loop, then calls `Stop()`. The `Use()` query allows external code to detect active load screen state.

## External Dependencies
- **OpenGL**: Conditional includes (`<OpenGL/gl.h>` macOS, `<GL/gl.h>` Linux/Windows)
- **OGL_Blitter**: Image blitting and texture tile management
- **ImageLoader.h**: `ImageDescriptor` class for loaded image data
- **SDL**: `SDL_Rect` for screen rectangles
- **cseries.h**: Core types (`rgb_color`, `uint32`, vector)
