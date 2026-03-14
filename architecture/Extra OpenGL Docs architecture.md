# Subsystem Overview

## Purpose
Reference documentation and code examples demonstrating advanced OpenGL rendering techniques: multitexturing with ARB extensions, rasterization-only mode, and software-controlled transformation pipelines for perspective-correct texture mapping. These are instructional materials, not part of the core Dingoo runtime.

## Key Files
| File | Role |
|------|------|
| `Extra OpenGL Docs/multitex.cpp` | ARB multitexture extension demonstration with dual-texture quad rendering and fallback to sequential single-pass mode |
| `Extra OpenGL Docs/rasonly.cpp` | Rasterization-only OpenGL example with manual vertex transformation pipeline (modelview ΓåÆ projection ΓåÆ perspective divide ΓåÆ viewport mapping) |
| `Extra OpenGL Docs/rasonly2.cpp` | Rasterization-only variant with application-level transformation matrices and per-vertex color support |

## Core Responsibilities
- Demonstrate ARB multitexture extension loading and dynamic function pointer resolution via `wglGetProcAddress`
- Show manual implementation of the transformation pipeline outside OpenGL's fixed-function stack (identity matrices for rasterization-only operation)
- Manage procedural texture generation (checkerboard, gradient) and GPU upload
- Handle keyboard and mouse input for interactive parameter control (panning, rotation, animation modes)
- Render textured quads with perspective correction and mipmap support
- Provide fallback behavior when extensions are unavailable (sequential texture passes instead of simultaneous multitexturing)

## Key Interfaces & Data Flow
- **Exports**: NoneΓÇöthese are standalone examples, not callable subsystem APIs
- **Dependencies**: OpenGL 1.2+ (GL/glut.h, GL/gl.h, GL/glu.h), Windows-specific `wglGetProcAddress` for extension discovery, GLUT event loop and bitmap fonts, C standard library (stdio, stdlib, string, assert)
- **No integration** with core Dingoo or Aleph One subsystems

## Runtime Role
Not integrated into runtime. These are compile-time examples requiring GLUT windowing and hardware OpenGL. Execution flow: GLUT event loop ΓåÆ keyboard/mouse input handling ΓåÆ matrix/transformation computation ΓåÆ texture binding ΓåÆ quad render with perspective-corrected coordinates ΓåÆ on-screen help display.

## Notable Implementation Details
- **multitex.cpp** conditionally compiles dual-texture rendering if `GL_ARB_multitexture` is available; otherwise falls back to two sequential single-texture passes on the same geometry
- **rasonly.cpp** and **rasonly2.cpp** both bypass OpenGL's transformation pipeline by setting identity modelview and orthographic projection, then computing vertex transformations in application code (manual perspective divide for correct texture coordinates)
- Perspective-correct texture mapping achieved using 4D homogeneous texture coordinates or explicit perspective-divide computations in vertex data
