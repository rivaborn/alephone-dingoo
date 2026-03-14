# Source_Files/RenderMain/OGL_Win32.cpp

## File Purpose
Windows-specific OpenGL extension loader for the Aleph One game engine. Detects and initializes OpenGL ARB extensions (multitexturing, texture compression) at startup by dynamically loading function pointers via SDL, enabling optional GPU features that may not be available on all systems.

## Core Responsibilities
- Query SDL for OpenGL extension function addresses using `SDL_GL_GetProcAddress`
- Detect multitexturing support by attempting to load `glActiveTextureARB` and `glClientActiveTextureARB`
- Set the global `has_multitex` flag based on extension availability
- Load the compressed texture function pointer `glCompressedTexImage2DARB`
- Warn (via stderr) if compressed texture extension is unavailable

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `GL_ACTIVETEXTUREARB_FUNC` | typedef | Function pointer type for `glActiveTextureARB(unsigned int)` |
| `GL_CLIENTACTIVETEXTUREARB_FUNC` | typedef | Function pointer type for `glClientActiveTextureARB(int)` |
| `PFNGLCOMPRESSEDTEXIMAGE2DARBPROC` | typedef | Standard OpenGL function pointer type for compressed texture loading |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `glActiveTextureARB_ptr` | `GL_ACTIVETEXTUREARB_FUNC` | global/static | Function pointer for active texture unit selection |
| `glClientActiveTextureARB_ptr` | `GL_CLIENTACTIVETEXTUREARB_FUNC` | global/static | Function pointer for client-side active texture unit selection |
| `glCompressedTexImage2DARB_ptr` | `PFNGLCOMPRESSEDTEXIMAGE2DARBPROC` | global/static | Function pointer for uploading compressed texture data |
| `has_multitex` | `int` | global/static | Flag indicating multitexturing support (0 or 1) |

## Key Functions / Methods

### setup_gl_extensions
- **Signature:** `void setup_gl_extensions()`
- **Purpose:** Initialize OpenGL extension function pointers and detect hardware capabilities at engine startup.
- **Inputs:** None
- **Outputs/Return:** None (modifies global state)
- **Side effects:** 
  - Writes to `glActiveTextureARB_ptr`, `glClientActiveTextureARB_ptr`, `glCompressedTexImage2DARB_ptr` globals
  - Sets `has_multitex` flag (0 if either multitex function fails, 1 if both succeed)
  - Outputs to stderr if compressed texture extension unavailable
- **Calls:** `SDL_GL_GetProcAddress` (3 times)
- **Notes:** 
  - Multitexturing is all-or-nothing: both ARB functions must load for `has_multitex=1`
  - Compressed texture failure is only logged, not fatal
  - Called during initialization before any rendering; order of loads suggests multitex is higher priority than compression

## Control Flow Notes
Initialization phase: called during engine startup (likely in the OpenGL context initialization sequence) before the main frame loop begins. Sets up capability flags and function pointers used throughout the rendering pipeline for optional GPU feature access.

## External Dependencies
- **System headers:** `<windows.h>` (Windows platform types)
- **OpenGL headers:** `<GL/gl.h>`, `<GL/glext.h>` (GL types and ARB extension definitions)
- **SDL:** `<SDL.h>` for `SDL_GL_GetProcAddress` (runtime OpenGL function lookup)
- **Standard library:** `<cstdio>` implied (fprintf, stderr)
- **Symbols defined elsewhere:** `glActiveTextureARB_ptr`, `glClientActiveTextureARB_ptr`, `glCompressedTexImage2DARB_ptr`, `has_multitex` (declared in `OGL_Win32.h`, instantiated here)
