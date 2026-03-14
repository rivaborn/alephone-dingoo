# Extra OpenGL Docs/multitex.cpp

## File Purpose
OpenGL sample program demonstrating ARB multitexturing by rendering a quad with two textures simultaneously. One texture is a checkerboard pattern, the other a radial gradient with color sectors; both can be panned or rotated independently via keyboard input. Falls back to single-texture rendering if the GL_ARB_multitexture extension is unavailable.

## Core Responsibilities
- Initialize OpenGL context and GLUT window management
- Generate and upload two textures (checkerboard and colored radial pattern) to GPU
- Manage dual texture coordinate sets for quad vertices
- Render quad with multitexturing or fallback single-texture mode
- Handle real-time texture transformations (pan/rotate) via texture matrix stack
- Process keyboard input for mode toggling and transformation control
- Display on-screen UI text (help, status, capability warnings)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Vertex` | struct | Position (x, y) and dual texture coordinates (u0, v0, u1, v1) for a quad vertex |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `tex0` | `unsigned char[]` | static | Raw RGB pixel data for checkerboard texture (128├ù128) |
| `tex1` | `unsigned char[]` | static | Raw RGBA pixel data for radial gradient texture (256├ù256) |
| `font` | `void*` | static | GLUT bitmap font handle |
| `windowWidth`, `windowHeight` | `int` | static | Window dimensions (600├ù600) |
| `angle` | `float` | static | Animation rotation angle (incremented each frame) |
| `multitexturing` | `int` | static | Boolean flag: 1 if GL_ARB_multitexture is available and enabled |
| `panTexture0` | `int` | static | Boolean flag: 0=rotate mode, 1=pan mode for texture 0 |
| `vtx` | `Vertex[4]` | static | Quad vertices with texture coordinates |
| `texNames` | `GLuint[2]` | static | OpenGL texture object handles |
| `glActiveTextureARB` | `PFNGLACTIVETEXTUREARBPROC` | static | Function pointer to glActiveTextureARB (or NULL if unavailable) |
| `glMultiTexCoord2fARB` | `PFNGLMULTITEXCOORD2FARBPROC` | static | Function pointer to glMultiTexCoord2fARB (or NULL if unavailable) |

## Key Functions / Methods

### MakeTextures
- **Purpose:** Generate two textures and upload them to GPU with different filtering and wrapping parameters.
- **Inputs:** None
- **Outputs/Return:** None (modifies `tex0`, `tex1`, and OpenGL texture state)
- **Side effects:** Allocates GPU texture memory via `glTexImage2D`, sets texture parameters (filtering, wrapping, environment mode). Populates `texNames[0]` and `texNames[1]`.
- **Calls:** `glGenTextures`, `glBindTexture`, `glTexImage2D`, `glTexParameteri`, `glTexEnvi`
- **Notes:** Texture 0 is nearest-neighbor (blocky checkerboard); Texture 1 is linear filtered. Texture 1 uses alpha channel computed from radial distance.

### Display
- **Purpose:** Render a frame with multitexturing (if available) or fallback single-texture dual-quad rendering.
- **Inputs:** None
- **Outputs/Return:** None (side effect: renders to framebuffer and swaps buffers)
- **Side effects:** Clears buffers, modifies texture matrices, sends vertex data to GPU, calls `DisplayInfo()` and `glutSwapBuffers()`.
- **Calls:** `glClear`, `glActiveTextureARB` (if multitexturing), `glMatrixMode`, `glLoadIdentity`, `glTranslatef`, `glRotatef`, `glBlendFunc`, `glEnable`, `glBegin`/`glEnd`, `glMultiTexCoord2fARB` or `glTexCoord2f`, `glVertex2f`, `DisplayInfo`, `glutSwapBuffers`
- **Notes:** Multitexture mode uses single quad with both texture units active; fallback mode renders two overlapping quads. Both paths apply same texture matrix transforms (pan or rotate).

### Reshape
- **Purpose:** Adjust viewport and projection matrix when window is resized.
- **Inputs:** `w`, `h` (new window width and height)
- **Outputs/Return:** None
- **Side effects:** Modifies OpenGL projection and modelview matrices.
- **Calls:** `glViewport`, `glMatrixMode`, `glLoadIdentity`, `glOrtho`, `glScalef`, `glTranslatef`
- **Notes:** Sets up orthographic projection with Y-axis inverted and origin at upper-left.

### Idle
- **Purpose:** Animation loop called when no events are pending.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Increments `angle` by 2.0, calls `Display()`.
- **Calls:** `Display`
- **Notes:** Drives continuous rotation of textures.

### GetGLProcs
- **Purpose:** Check for GL_ARB_multitexture extension and load function pointers.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies `multitexturing` flag and function pointers; calls `OutputDebugString` if extension unavailable.
- **Calls:** `glGetString`, `strstr`, `wglGetProcAddress`, `OutputDebugString`
- **Notes:** If extension not found, disables multitexturing; otherwise loads `glActiveTextureARB` and `glMultiTexCoord2fARB`.

### Keyboard
- **Purpose:** Handle keyboard events for mode and transformation toggles.
- **Inputs:** `key` (ASCII character), `x`, `y` (mouse position, unused)
- **Outputs/Return:** None
- **Side effects:** Modifies `multitexturing` or `panTexture0`, calls `Init()`, may call `exit()`.
- **Calls:** `Init`, `exit`
- **Notes:** Space toggles multitexturing (if available), P=pan, R=rotate, Tab=toggle pan/rotate, Q=quit.

### InitVertices
- **Purpose:** Set up quad vertex positions and texture coordinates.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Populates global `vtx[4]` array.
- **Calls:** None
- **Notes:** Creates unit quad [0,0] to [1,1] with offset and scale (QUAD_SIZE, QUAD_X_OFFSET, QUAD_Y_OFFSET); both texture coordinates map to same normalized positions.

### Init
- **Purpose:** Set up texture units (multitexture mode only).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Activates texture units, binds textures, sets environment mode.
- **Calls:** `glActiveTextureARB`, `glBindTexture`, `glTexEnvi`
- **Notes:** Only active if `multitexturing` is true; sets both units to GL_MODULATE mode.

### OutputString, OutputShadowedString, DisplayInfo
- **Purpose:** Render help text and status overlay on screen.
- **Inputs:** OutputString: `x`, `y`, `string`; DisplayInfo: none
- **Outputs/Return:** None (side effect: renders text)
- **Side effects:** Modify projection/modelview matrices, call `glutBitmapCharacter`.
- **Calls:** `glRasterPos2f`, `glutBitmapCharacter`, `glColor3f`, `OutputString`, `sprintf`, `glViewport`, `glMatrixMode`, `glLoadIdentity`, `gluOrtho2D`
- **Notes:** OutputShadowedString renders a black shadow then yellow text for contrast.

### main
- **Purpose:** Application entry point; initialize GLUT, OpenGL, and enter event loop.
- **Inputs:** `argc`, `argv` (command line arguments, unused)
- **Outputs/Return:** 0 on exit
- **Side effects:** Initializes GLUT, creates window, registers callbacks, enters infinite loop.
- **Calls:** `glutInit`, `glutInitDisplayMode`, `glutInitWindowSize`, `glutCreateWindow`, `glutSetWindowTitle`, `glGetString`, `sprintf`, `GetGLProcs`, `MakeTextures`, `InitVertices`, `Init`, `glutDisplayFunc`, `glutReshapeFunc`, `glutKeyboardFunc`, `glutIdleFunc`, `glutMainLoop`
- **Notes:** Requires `InitVertices` to be called before rendering; sets up callbacks for Display, Reshape, Keyboard, and Idle.

## Control Flow Notes
**Initialization:** `main()` ΓåÆ `glutInit()` / window creation ΓåÆ `GetGLProcs()` (check extensions) ΓåÆ `MakeTextures()` (upload textures) ΓåÆ `InitVertices()` (setup quad) ΓåÆ `Init()` (bind texture units) ΓåÆ `glutMainLoop()`.

**Frame/Event Loop:** `glutMainLoop()` continuously checks for events and invokes callbacks. In idle periods, `Idle()` increments `angle` and calls `Display()`. On reshape, `Reshape()` updates projection. On keyboard, `Keyboard()` updates state and re-initializes texture units.

**Render Path:** `Display()` clears buffers, then either (a) renders one quad with both textures active (multitexture path), or (b) renders two quads sequentially with different textures and blend modes (fallback path). Both paths apply the same texture matrix transforms (translate + rotate for tex1, translate + rotate or pan for tex0).

## External Dependencies
- **GLUT (GL/glut.h):** Window creation, event loop, bitmap font rendering, buffer swap.
- **OpenGL Extensions (glext.h, wglGetProcAddress):** GL_ARB_multitexture function pointers; defines `PFNGLACTIVETEXTUREARBPROC`, `PFNGLMULTITEXCOORD2FARBPROC`.
- **Standard C (stdio.h):** `sprintf`, `strlen`, `strstr`.
- **Defined elsewhere:** GLUT callbacks (`glutDisplayFunc`, etc.), OpenGL context (created by GLUT), OpenGL immediate-mode API (`glBegin`/`glEnd`, etc.).
