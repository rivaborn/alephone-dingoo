# Source_Files/ModelView/ModelRenderer.cpp

## File Purpose
Implements 3D model rendering for the Aleph One game engine with support for multi-pass shading, depth-sorted polygon rendering (when Z-buffer unavailable), and callback-driven texture/lighting management.

## Core Responsibilities
- Render 3D models with one or multiple shader passes
- Perform depth-sorting of triangles by centroid when Z-buffer is unavailable
- Set up OpenGL state (vertex arrays, textures, colors, lighting) for each render pass
- Optimize rendering paths based on shader separability and Z-buffer availability
- Manage temporary buffers for centroid depths, sorted indices, and lighting colors

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ModelRenderShader` | struct | Encapsulates shader rendering state: flags, texture callback, lighting callback, and associated data pointers |
| `IndexedCentroidDepth` | struct | Pairs triangle index with depth value for sorting; implements `operator<` to sort farthest-to-nearest |
| `Model3D` | class (external) | Container for vertex positions, indices, texture coordinates, normals, and colors |

## Global / File-Static State
None.

## Key Functions / Methods

### Render
- **Signature:** `void Render(Model3D& Model, ModelRenderShader *Shaders, int NumShaders, int NumSeparableShaders, bool Use_Z_Buffer)`
- **Purpose:** Main entry point for rendering a 3D model with one or multiple shader passes, with optional depth-sorting of polygons.
- **Inputs:**
  - `Model`: 3D model containing geometry, colors, normals, texture coordinates
  - `Shaders`: array of shader descriptors
  - `NumShaders`: total shaders in array
  - `NumSeparableShaders`: count of shaders that can be rendered in a single pass (requires Z-buffer); assumed to be first shaders in array
  - `Use_Z_Buffer`: whether depth-testing is enabled
- **Outputs/Return:** None (side effects on OpenGL state and model rendering)
- **Side effects:**
  - Modifies `IndexedCentroidDepths`, `SortedVertIndices`, `ExtLightColors` member arrays
  - Calls OpenGL functions (`glEnableClientState`, `glVertexPointer`, `glDrawElements`)
  - Calls `SetupRenderPass` for each pass
- **Calls:** `SetupRenderPass`, `std::sort`, OpenGL state functions
- **Notes:**
  - Early returns if no shaders, null shader array, or empty model
  - Sets `NumSeparableShaders = 0` if Z-buffer disabled (all shaders must be depth-sorted)
  - Optimization: if all shaders are separable, renders all at once without depth-sorting
  - Computes triangle centroids by averaging vertex positions, projects onto `ViewDirection` for depth
  - Sorts centroids and reorders triangle indices once for all separable shaders
  - Non-separable shaders rendered triangle-by-triangle to maintain correct blending order

### SetupRenderPass
- **Signature:** `void SetupRenderPass(Model3D& Model, ModelRenderShader& Shader)`
- **Purpose:** Configure OpenGL state for a single render pass (texture coordinates, colors, lighting).
- **Inputs:**
  - `Model`: provides texture coordinates, normals, positions, colors
  - `Shader`: flags and callbacks for texture and lighting setup
- **Outputs/Return:** None (side effects on OpenGL state)
- **Side effects:**
  - Enables/disables GL_TEXTURE_2D and GL_TEXTURE_COORD_ARRAY
  - Enables/disables GL_COLOR_ARRAY
  - Populates `ExtLightColors` if external lighting callback is used
  - Calls `Shader.TextureCallback` and conditionally `Shader.LightingCallback`
- **Calls:** OpenGL state functions, lighting and texture callbacks
- **Notes:**
  - Blends external lighting colors (from callback) with model colors if both present
  - Handles 3-channel (RGB) and 4-channel (RGBA with alpha) lighting colors separately
  - Falls back to plain model colors if external lighting unavailable

### Clear
- **Signature:** `void Clear()`
- **Purpose:** Reset temporary rendering buffers.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `IndexedCentroidDepths`, `SortedVertIndices`, `ExtLightColors` vectors
- **Notes:** Called to release memory between renders or at shutdown

## Control Flow Notes
Called during the frame render phase to draw 3D models. The `ViewDirection` member (set externally) controls depth-sorting direction; rendering pipeline configuration (Z-buffer enabled/disabled, separable shaders) determines whether depth-sorting is performed.

## External Dependencies
- **OpenGL:** `glEnableClientState`, `glVertexPointer`, `glDrawElements`, `glEnable`, `glDisable`, `glTexCoordPointer`, `glColorPointer`, GL constants
- **STL:** `std::sort`, `std::vector`
- **Model3D:** external type with `.Positions`, `.VertIndices`, `.TxtrCoords`, `.Normals`, `.Colors` arrays and accessor methods (`.PosBase()`, `.VIBase()`, `.TCBase()`, `.NormBase()`, `.ColBase()`, `.NumVI()`)
- **cseries.h:** `TEST_FLAG` macro, platform/compiler abstraction
- **ModelRenderer.h:** class definition, `IndexedCentroidDepth`, `ModelRenderShader`
