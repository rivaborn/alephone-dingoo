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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| quadV | GLfloat[4][4] array | 4 quad vertices with homogeneous w coordinates |
| quadC | GLfloat[4][3] array | RGB colors per vertex |
| quadT | GLfloat[4][2] array | Texture coordinates per vertex |
| ModelView | GLfloat[16] array | 4├ù4 ModelView transformation matrix (row-major) |
| Projection | GLfloat[16] array | 4├ù4 Projection transformation matrix (row-major) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| motion | GLboolean | global | Toggle continuous animation via idle callback |
| rot | GLfloat | global | Current rotation angle (degrees) |
| speed | GLfloat | global | Rotation speed multiplier |
| texWidth, texHeight | int | global | Texture dimensions (128├ù128) |

## Key Functions / Methods

### Transform
- Signature: `static void Transform(GLfloat *matrix, GLfloat *in, GLfloat *out)`
- Purpose: Multiply a 4├ù4 matrix by a 4D vector
- Inputs: matrix (4├ù4 row-major), in (4D vertex)
- Outputs/Return: out (transformed 4D vertex)
- Calls: None (inline computation)
- Notes: Row-major layout (0\*4+ii indexing)

### DoTransform
- Signature: `static void DoTransform(GLfloat *in, GLfloat *out)`
- Purpose: Full vertex transformation pipeline (ModelView ΓåÆ Projection)
- Inputs: in (object-space vertex)
- Outputs/Return: out (clip-space vertex)
- Calls: Transform (├ù2)
- Notes: Comments indicate lighting would be inserted between ModelView and Projection transforms

### UpdateModelView
- Signature: `void UpdateModelView(void)`
- Purpose: Compute ModelView matrix from camera (gluLookAt) and current rotation
- Side effects: Updates global ModelView via glGetFloatv
- Calls: glPushMatrix, glLoadIdentity, gluLookAt, glRotatef, glGetFloatv, glPopMatrix

### UpdateProjection
- Signature: `void UpdateProjection(void)`
- Purpose: Compute Projection matrix (45┬░ FOV, 1:1 aspect, 1ΓÇô100 depth range)
- Side effects: Updates global Projection; temporarily modifies GL_PROJECTION matrix mode
- Calls: glMatrixMode, glPushMatrix, glLoadIdentity, gluPerspective, glGetFloatv, glPopMatrix

### setCheckedTexture
- Signature: `static void setCheckedTexture(void)`
- Purpose: Generate and upload a checkerboard texture via gluBuild2DMipmaps
- Side effects: Allocates and frees temporary buffer; modifies GPU texture state
- Calls: malloc, free, gluBuild2DMipmaps
- Notes: Uses XOR pattern (i^j & 8) to create squares; allocates RGBA worst-case

### Redraw
- Signature: `void Redraw(void)`
- Purpose: Render the quad with per-vertex colors and texture coordinates
- Side effects: Clears framebuffer, swaps buffers
- Calls: glClear, glBegin/glEnd, DoTransform, glColor3fv, glTexCoord2fv, glVertex4fv, glutSwapBuffers
- Notes: Specifies homogeneous w in glVertex4fv to enable hardware perspective correction

### Motion
- Signature: `void Motion(void)`
- Purpose: Idle callback that rotates the model and redraws
- Side effects: Updates global rot (wraps at 360┬░), triggers redraw
- Calls: UpdateModelView, Redraw

### Key / Button
- Handle ESC (exit), 'm' (toggle motion), '+'/'-' (speed), and mouse clicks (manual rotation)

## Control Flow Notes
1. **Init phase** (main ΓåÆ Init): glClearColor, texture setup, identity matrices, UpdateProjection/UpdateModelView
2. **Event loop** (glutMainLoop): Dispatches to Redraw, Motion (idle), Key, Button, Reshape
3. **Frame rendering** (Redraw): Clear, transform each vertex via DoTransform, emit quad, swap
4. **Rotation update** (Motion/Button): Modify rot, call UpdateModelView, call Redraw

## External Dependencies
- **OpenGL/GLUT**: GL/gl.h, GL/glut.h, GL/glu.h ΓÇö all graphics state and rendering
- **Standard C**: stdio.h, stdlib.h, string.h, assert.h
