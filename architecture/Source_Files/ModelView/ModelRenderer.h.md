# Source_Files/ModelView/ModelRenderer.h

## File Purpose
Defines the `ModelRenderer` class for rendering 3D model geometry with optional Z-buffering and polygon depth-sorting. Supports multipass rendering via shader callbacks for texturing and per-vertex lighting. Part of the Aleph One engine (Marathon remake).

## Core Responsibilities
- Render 3D models with configurable per-triangle or per-vertex shading via callbacks
- Manage depth-sorting of model triangles by centroid when Z-buffer unavailable
- Support separable vs. non-separable shader passes (with Z-buffer optimization)
- Maintain persistent scratch buffers (centroid depths, sorted indices, lighting colors) to avoid re-allocation
- Apply external lighting and semitransparency flags to vertex data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ModelRenderShader` | struct | Encodes rendering pass: flags (textured/colored/lit), texture callback, lighting callback with vertex data |
| `IndexedCentroidDepth` | struct | Pairs polygon index with depth for reverse-sorted (farthest-to-nearest) polygon ordering |
| `ModelRenderer` | class | Main renderer with `Render()` and buffer management |

## Global / File-Static State
None.

## Key Functions / Methods

### Render
- **Signature:** `void Render(Model3D& Model, ModelRenderShader *Shaders, int NumShaders, int NumSeparableShaders, bool Use_Z_Buffer)`
- **Purpose:** Execute multipass rendering of a 3D model using provided shader array
- **Inputs:**
  - `Model`: 3D geometry (vertices, normals, indices, bones, frames)
  - `Shaders`: Array of `ModelRenderShader` configuration objects
  - `NumShaders`: Total shaders to use
  - `NumSeparableShaders`: Count of all-or-nothing shaders rendered first (only if Z-buffer present)
  - `Use_Z_Buffer`: If false, performs depth-sort on centroid instead
- **Outputs/Return:** None (draws to GL context)
- **Side effects:** Populates `IndexedCentroidDepths`, `SortedVertIndices`, `ExtLightColors` buffers; invokes callback functions
- **Calls:** `SetupRenderPass()` (visible signature only)
- **Notes:** Semitransparent shaders are nonseparable. Without Z-buffer, all shaders are depth-sorted together.

### SetupRenderPass
- **Signature:** `void SetupRenderPass(Model3D& Model, ModelRenderShader& Shader)` (private)
- **Purpose:** Configure rendering state for a single shader pass (texture binding, lighting setup)
- **Inputs:** Model and shader configuration
- **Outputs/Return:** None
- **Side effects:** Invokes texture and lighting callbacks; modifies GL state
- **Notes:** Implementation not visible in header; details inferable only from callback signatures

### Clear
- **Signature:** `void Clear()`
- **Purpose:** Reset persistent scratch buffers for fresh rendering
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `IndexedCentroidDepths`, `SortedVertIndices`, `ExtLightColors` vectors

## Control Flow Notes
Typical flow: construct `ModelRenderer`, populate `ViewDirection` (world-space camera direction for depth-sort), call `Render()` with shader array. If Z-buffer available, separable shaders render in single pass; non-separable shaders require separate passes. Without Z-buffer, polygon centroids are projected onto view direction, sorted, and rendered back-to-front. Callbacks execute per-pass or per-vertex depending on configuration.

## External Dependencies
- `#include "Model3D.h"` ΓÇö 3D geometry container (vertices, normals, bones, frames, transforms)
- OpenGL (`GL/gl.h` or platform equivalent via Model3D.h) ΓÇö GLfloat, GLushort, GLfloat arrays for rendering
