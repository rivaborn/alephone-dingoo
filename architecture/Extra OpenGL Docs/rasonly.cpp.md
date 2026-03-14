# Extra OpenGL Docs/rasonly.cpp

## File Purpose

Educational demonstration of OpenGL rasterization-only rendering with perspective-correct texture mapping. The application manually handles vertex transformations in CPU code while using OpenGL purely for rasterization, targeting low-end 3D accelerators that lack hardware transformation pipelines.

## Core Responsibilities

- Demonstrate rasterization-only OpenGL setup (identity projection, manual CPU transforms)
- Perform vertex transformation pipeline: modelview ΓåÆ projection ΓåÆ perspective divide ΓåÆ viewport mapping
- Generate and manage a checkerboard texture with mipmapping
- Render a perspective-correct textured quad
- Handle user input (keyboard, mouse, window resize) to manipulate the scene
- Manage 4├ù4 transformation matrices (modelview, projection, viewport)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `quadV` | float array `[4][4]` | Quad vertices in homogeneous object coordinates |
| `quadC` | float array `[4][3]` | Per-vertex RGB colors |
| `quadT` | float array `[4][2]` | Per-vertex texture coordinates (normalized 0ΓÇô1) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `motion` | GLboolean | static | Enables/disables continuous animation |
| `rot` | GLfloat | static | Current rotation angle (degrees) around Y-axis |
| `ModelView` | GLfloat[16] | static | 4├ù4 modelview transformation matrix |
| `Projection` | GLfloat[16] | static | 4├ù4 perspective projection matrix |
| `Viewport` | GLfloat[4] | static | Viewport bounds: [x, y, width, height] |
| `texWidth`, `texHeight` | int | static | Texture dimensions (128├ù128) |

## Key Functions / Methods

### `setCheckedTexture()`
- **Purpose:** Generate a checkerboard texture pattern and upload to GPU via OpenGL mipmapping.
- **Inputs:** None (uses global `texWidth`, `texHeight`).
- **Outputs/Return:** None.
- **Side effects:** Allocates heap memory; calls `gluBuild2DMipmaps()` to upload texture; frees memory.
- **Calls:** `malloc()`, `gluBuild2DMipmaps()`, `free()`.
- **Notes:** Creates 4-channel RGBA with white/dark squares based on bit pattern `(i ^ j) & 8`.

### `Transform(matrix, in, out)`
- **Purpose:** Multiply a 4D homogeneous vector by a 4├ù4 column-major matrix.
- **Inputs:** `matrix` (4├ù4 float array), `in` (4D vector).
- **Outputs/Return:** `out` (4D result vector).
- **Side effects:** Writes to `out` array.
- **Calls:** None (inline loop).
- **Notes:** Column-major order: `out[ii] = ╬ú(in[j] * matrix[j*4+ii])`.

### `DoTransform(in, out)`
- **Purpose:** Complete vertex transformation pipeline: object ΓåÆ eye (modelview) ΓåÆ clip (projection) ΓåÆ NDC (perspective divide) ΓåÆ window coordinates.
- **Inputs:** `in` (object-space homogeneous vertex).
- **Outputs/Return:** `out` (window-space vertex; `out[3]` stores `1/w` for texture perspective correction).
- **Side effects:** Reads global `ModelView`, `Projection`, `Viewport` matrices.
- **Calls:** `Transform()` (├ù2 for modelview and projection).
- **Notes:** Stores inverted W (`1/w`) in `out[3]` for use in texture coordinate perspective correction via `glTexCoord4f()`.

### `UpdateModelView()`
- **Purpose:** Recalculate modelview matrix based on current rotation angle and camera position.
- **Inputs:** None (uses global `rot`).
- **Outputs/Return:** None.
- **Side effects:** Updates global `ModelView` via `glGetFloatv()`.
- **Calls:** `gluLookAt()`, `glRotatef()`, `glGetFloatv()`, `glPushMatrix()`, `glPopMatrix()`, `glLoadIdentity()`.
- **Notes:** Uses OpenGL matrix stack for convenience rather than manual math.

### `InitMatrices()`
- **Purpose:** Set up projection and initial modelview matrices using OpenGL calls, then extract them for CPU use.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Updates global `Projection`, `ModelView`; modifies GL matrix state.
- **Calls:** `gluPerspective()`, `glGetFloatv()`, `UpdateModelView()`.

### `Init()`
- **Purpose:** Initialize all OpenGL state: clear color, texture parameters, enable depth testing, upload checkerboard texture, set up matrices.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Configures GL state; allocates and uploads texture; populates global matrices.
- **Calls:** `glClearColor()`, `glTexParameteri()`, `glHint()`, `glEnable()`, `setCheckedTexture()`, `InitMatrices()`.

### `Redraw()`
- **Purpose:** Frame rendering: clear buffers, transform and render textured quad with perspective-correct texture coordinates.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Renders to framebuffer; calls `glutSwapBuffers()`.
- **Calls:** `glClear()`, `glBegin()` / `glEnd()`, `DoTransform()`, `glColor3fv()`, `glTexCoord4f()`, `glVertex3fv()`, `glutSwapBuffers()`.
- **Notes:** Per vertex: transforms from object to window space, scales texture coords by `1/w`, and submits 3D vertex (perspective divide already applied).

### `Motion()`
- **Purpose:** Idle callback for continuous animation; increment rotation and redraw.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Updates global `rot`; calls `UpdateModelView()` and `Redraw()`.
- **Calls:** `UpdateModelView()`, `Redraw()`.

### `Key(unsigned char key, int x, int y)`
- **Purpose:** Keyboard input handler.
- **Inputs:** `key` (ASCII code), `x`, `y` (cursor position, unused).
- **Outputs/Return:** None.
- **Side effects:** ESC exits; 'm' toggles animation via `glutIdleFunc()`.
- **Calls:** `glutIdleFunc()`, `exit()`.

### `Button(int button, int state, int x, int y)`
- **Purpose:** Mouse click handler to rotate via left/right buttons.
- **Inputs:** `button` (GLUT_LEFT/RIGHT_BUTTON), `state` (GLUT_UP/DOWN), `x`, `y` (unused).
- **Outputs/Return:** None.
- **Side effects:** Left-click: `rot -= 15┬░`; right-click: `rot += 15┬░`. Updates and redraws.
- **Calls:** `UpdateModelView()`, `Redraw()`.

### `Reshape(int width, int height)`
- **Purpose:** Window resize callback; set up orthographic projection for rasterization-only and update viewport.
- **Inputs:** `width`, `height` (new window dimensions).
- **Outputs/Return:** None.
- **Side effects:** Sets GL projection and viewport; updates global `Viewport[4]`.
- **Calls:** `glMatrixMode()`, `glLoadIdentity()`, `glOrtho()`, `glViewport()`, `glDepthRange()`.
- **Notes:** Uses orthographic projection (0,0) to (width,height) with depth [0,1]; no perspective (identity w).

### `main(int argc, char *argv[])`
- **Purpose:** Program entry point; initialize GLUT, set up callbacks, and enter event loop.
- **Inputs:** Standard C command-line arguments (unused).
- **Outputs/Return:** 0 on exit.
- **Side effects:** Creates window; registers display/reshape/input callbacks; blocks in `glutMainLoop()`.
- **Calls:** `glutInit()`, `glutInitDisplayMode()`, `glutInitWindowSize()`, `glutCreateWindow()`, `Init()`, `glutDisplay/Reshape/KeyboardFunc()`, `glutMouseFunc()`, `glutIdleFunc()`, `glutMainLoop()`.

## Control Flow Notes

- **Initialization:** `main()` ΓåÆ GLUT setup ΓåÆ `Init()` ΓåÆ `InitMatrices()`.
- **Frame loop:** GLUT event loop (blocking in `glutMainLoop()`).
  - Idle callback: `Motion()` increments rotation, updates matrix, calls `Redraw()`.
  - Input callbacks: `Key()`, `Button()`, `Reshape()` may update `rot` or viewport, then trigger redraw.
  - Display callback: `Redraw()` transforms geometry and renders.
- **Per-frame vertex processing:** `DoTransform()` runs on each quad vertex to compute window coordinates and `1/w` for texture perspective correction.

## External Dependencies

- **OpenGL/GLUT:** `GL/gl.h`, `GL/glu.h`, `GL/glut.h` ΓÇö windowing, matrix ops, texture, rendering.
- **Standard C:** `stdio.h`, `stdlib.h`, `string.h`, `assert.h` ΓÇö memory, utilities.
- **Platform-specific:** `#ifdef WIN32` for calling convention (`__cdecl`).
