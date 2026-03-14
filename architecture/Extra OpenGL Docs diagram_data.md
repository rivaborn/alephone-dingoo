# Extra OpenGL Docs/multitex.cpp
## File Purpose
Demonstrates OpenGL multitexturing using the GL_ARB_multitexture extension. Renders a single quad with two independently animated textures and falls back to sequential single-texture rendering if the extension is unavailable.

## Core Responsibilities
- Generate and manage two 2D textures (checkerboard and multicolor gradient)
- Load and invoke ARB multitexture extension function pointers via `wglGetProcAddress`
- Render a quad with either multitexturing or dual single-texture passes
- Handle interactive keyboard controls for texture panning/rotation modes
- Manage viewport/projection setup and window events via GLUT
- Display on-screen help text and feature availability

## External Dependencies
- `GL/glut.h` ΓÇö GLUT windowing and bitmap font rendering
- `glext.h` ΓÇö OpenGL extension type definitions (e.g., PFNGLACTIVETEXTUREARBPROC)
- `stdio.h` ΓÇö sprintf, strlen
- `wglGetProcAddress` (Windows-specific) ΓÇö runtime function pointer lookup

# Extra OpenGL Docs/rasonly.cpp
## File Purpose
Demonstrates using OpenGL for rasterization-only (with application-controlled transformations and lighting) while achieving perspective-correct texture mapping. Shows how to bypass OpenGL's transform pipeline by manually performing vertex transformations and using 4D texture coordinates for perspective correction.

## Core Responsibilities
- Demonstrate rasterization-only OpenGL setup (identity modelview, orthographic projection mapping to screen space)
- Implement full vertex transformation pipeline: modelview ΓåÆ projection ΓåÆ perspective divide ΓåÆ viewport mapping
- Generate and manage a procedural checkerboard texture with mipmaps
- Handle real-time rotation via idle callback and mouse input
- Render a single textured quad with perspective-corrected texture coordinates
- Manage window resizing and OpenGL state initialization

## External Dependencies
- **OpenGL/GLU headers:** `<GL/gl.h>`, `<GL/glu.h>`, `<GL/glut.h>` ΓÇö window system, rendering, matrix utilities.
- **C standard library:** `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<assert.h>` ΓÇö memory, I/O.
- **Defined elsewhere:** OpenGL command functions (glClear, glVertex3fv, etc.), GLUT event loop and callbacks, GLU matrix helpers (gluLookAt, gluPerspective, gluBuild2DMipmaps).

# Extra OpenGL Docs/rasonly2.cpp
## File Purpose
Demonstrates OpenGL used purely for rasterization (hardware pixel drawing) while the application handles transformations and lighting in software. Shows perspective-correct rendering and depth buffering with identity OpenGL matrices to bypass the fixed-function transformation pipeline.

## Core Responsibilities
- Set up OpenGL in rasterization-only mode with identity matrices
- Compute and manage application-level transformation matrices (ModelView, Projection)
- Generate and upload a checkerboard texture to the GPU
- Transform vertices from object to window coordinates manually
- Render a rotating textured quad with per-vertex colors
- Handle user input (keyboard and mouse) to control rotation and animation

## External Dependencies
- **OpenGL/GLUT**: GL/gl.h, GL/glut.h, GL/glu.h ΓÇö all graphics state and rendering
- **Standard C**: stdio.h, stdlib.h, string.h, assert.h


