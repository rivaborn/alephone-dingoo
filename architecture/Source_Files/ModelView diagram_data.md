# Source_Files/ModelView/Dim3_Loader.cpp
## File Purpose
Loads and parses Dim3 3D model format XML files, converting vertex/bone/animation data into the engine's Model3D structure. Handles skeletal mesh hierarchy, rigged vertex blending, and animation frame sequences derived from the Dim3 engine format.

## Core Responsibilities
- Parse Dim3 XML model files through XML_Configure framework
- Load and index vertices with position, normals, and dual-bone blend weights
- Build bone hierarchy from parent-child relationships; reorder bones via tree traversal
- Parse animation frames (poses) with per-bone rotations and offsets
- Parse animation sequences (animations) as ordered frame lists
- Remap vertex bone indices to canonical bone order
- Propagate vertex normals into the Model3D structure
- Convert Dim3 angle conventions to engine units

## External Dependencies
- **cseries.h** ΓÇô Engine types, macros (FULL_CIRCLE, NORMALIZE_ANGLE, UNONE, NONE, objlist_clear, obj_clear, obj_copy)
- **Dim3_Loader.h** ΓÇô LoadModel_Dim3 signature, SetDebugOutput_Dim3
- **world.h** ΓÇô Angle constants/macros (FULL_CIRCLE, NORMALIZE_ANGLE)
- **XML_Configure.h** ΓÇô Base parser class (DoParse, Buffer, BufLen, LastOne, CurrentElement)
- **XML_ElementParser.h** ΓÇô Element parser base (StringsEqual, ReadFloatValue, ReadUInt16Value, UnrecognizedTag)
- **FileHandler.h** ΓÇô FileSpecifier, OpenedFile (Open, GetLength, Read, GetName)
- **Model3D.h** ΓÇô Model3D, Model3D_VertexSource, Model3D_Bone, Model3D_Frame, Model3D_SeqFrame (VtxSources, Bones, Frames, SeqFrames, BoundingBox, NormSources, etc.)
- **Standard:** `<math.h>`, `<stdio.h>`, `<windows.h>` (Windows only)
- **Expat** (via XML_Configure.h) ΓÇô XML parser engine

# Source_Files/ModelView/Dim3_Loader.h
## File Purpose
Header for the Dim3 3D model format loader. Declares the primary loading function and debug configuration interface for parsing Dim3 models (supports multi-file models via multiple passes).

## Core Responsibilities
- Declare multi-pass model loading control (first pass vs. subsequent passes)
- Provide the main `LoadModel_Dim3()` entry point for loading models from files
- Configure debug output destination for loader status messages

## External Dependencies
- `<stdio.h>` ΓÇö FILE type for debug output
- `Model3D.h` ΓÇö Model3D struct (target for loading)
- `FileHandler.h` ΓÇö FileSpecifier abstraction (file path handling)

# Source_Files/ModelView/Model3D.cpp
## File Purpose

Implements 3D model storage and transformation for the Aleph One game engine. Handles skeletal animation with bones/frames, vertex position computation, normal calculation/normalization, and bounding box management for OpenGL rendering.

## Core Responsibilities

- **Skeletal animation:** Build bone transformation matrices, apply to vertex sources with optional interpolation
- **Vertex position computation:** Three modes (neutral static, frame-based, sequence-based animation)
- **Normal processing:** Normalize, reverse, or recompute normals; optionally split vertices for smooth shading with hard edges
- **Bone hierarchy:** Manage parent-child relationships using push/pop stack during frame-to-position traversal
- **Dual-bone blending:** Support weighted blending of two bones per vertex
- **Bounding box:** Compute and render axis-aligned bounding box for debugging
- **Transformation matrices:** Apply affine 3D transforms (rotation + translation) to points and vectors

## External Dependencies

- **VecOps.h:** Vector operation templates (`VecCopy`, `VecSub`, `ScalarProd`, `VecAddTo`, `VecScalarMultTo`)
- **cseries.h:** Compatibility layer (`objlist_copy`, `objlist_clear` macros; cross-platform type definitions)
- **world.h:** Engine math (`angle`, `cosine_table`, `sine_table`, `TRIG_MAGNITUDE`, `build_trig_tables()`, `NORMALIZE_ANGLE`)
- **Model3D.h:** Data structure definitions (`Model3D`, `Model3D_Transform`, `Model3D_Bone`, `Model3D_Frame`, etc.)
- **OGL_Setup.h:** OpenGL utilities (`SglColor3fv()` color setter)
- **GL/gl.h:** OpenGL C API (`glDisable`, `glEnableClientState`, `glVertexPointer`, `glDrawElements`)
- **Defined elsewhere:** `UNONE` (sentinel value, likely max unsigned), `TEST_FLAG`, `MAX`, `MIN`, `sqrt` (math library)

# Source_Files/ModelView/Model3D.h
## File Purpose
Defines data structures for storing 3D model geometry and skeletal animation data in an OpenGL-friendly format. This is the core 3D asset container for the Aleph One game engine, supporting vertex data, bone hierarchies, animation frames, and sequences.

## Core Responsibilities
- Store vertex geometry (positions, texture coordinates, normals, colors) in contiguous OpenGL-friendly arrays
- Define skeletal animation structures (bones, frames, vertex blending)
- Support animation sequences with crossfading between frames
- Manage transformation matrices for model-space to render-space conversion
- Calculate and cache bounding boxes for culling/debugging
- Provide methods to compute vertex positions at neutral pose, specific animation frames, or sequences
- Handle normal vector processing (recalculation, reversal, smoothing)

## External Dependencies
- **OpenGL:** Platform-conditional includes (`<OpenGL/gl.h>` on macOS, `<GL/gl.h>` elsewhere; Windows-specific `<wingdi.h>` for compatibility).
- **STL:** `<vector>` for all dynamic arrays.
- **Global:** Assumes `BuildTrigTables()` initializes Marathon-format trig lookups elsewhere (e.g., `world.h`).

# Source_Files/ModelView/ModelRenderer.cpp
## File Purpose
Implements 3D model rendering for the Aleph One game engine with support for multi-pass shading, depth-sorted polygon rendering (when Z-buffer unavailable), and callback-driven texture/lighting management.

## Core Responsibilities
- Render 3D models with one or multiple shader passes
- Perform depth-sorting of triangles by centroid when Z-buffer is unavailable
- Set up OpenGL state (vertex arrays, textures, colors, lighting) for each render pass
- Optimize rendering paths based on shader separability and Z-buffer availability
- Manage temporary buffers for centroid depths, sorted indices, and lighting colors

## External Dependencies
- **OpenGL:** `glEnableClientState`, `glVertexPointer`, `glDrawElements`, `glEnable`, `glDisable`, `glTexCoordPointer`, `glColorPointer`, GL constants
- **STL:** `std::sort`, `std::vector`
- **Model3D:** external type with `.Positions`, `.VertIndices`, `.TxtrCoords`, `.Normals`, `.Colors` arrays and accessor methods (`.PosBase()`, `.VIBase()`, `.TCBase()`, `.NormBase()`, `.ColBase()`, `.NumVI()`)
- **cseries.h:** `TEST_FLAG` macro, platform/compiler abstraction
- **ModelRenderer.h:** class definition, `IndexedCentroidDepth`, `ModelRenderShader`

# Source_Files/ModelView/ModelRenderer.h
## File Purpose
Defines the `ModelRenderer` class for rendering 3D model geometry with optional Z-buffering and polygon depth-sorting. Supports multipass rendering via shader callbacks for texturing and per-vertex lighting. Part of the Aleph One engine (Marathon remake).

## Core Responsibilities
- Render 3D models with configurable per-triangle or per-vertex shading via callbacks
- Manage depth-sorting of model triangles by centroid when Z-buffer unavailable
- Support separable vs. non-separable shader passes (with Z-buffer optimization)
- Maintain persistent scratch buffers (centroid depths, sorted indices, lighting colors) to avoid re-allocation
- Apply external lighting and semitransparency flags to vertex data

## External Dependencies
- `#include "Model3D.h"` ΓÇö 3D geometry container (vertices, normals, bones, frames, transforms)
- OpenGL (`GL/gl.h` or platform equivalent via Model3D.h) ΓÇö GLfloat, GLushort, GLfloat arrays for rendering

# Source_Files/ModelView/QD3D_Loader.cpp
## File Purpose
Loads 3D models in QuickDraw 3D (QD3D) / Quesa format and tessellates them into triangle meshes with deduplicated vertices. Uses a custom fake renderer to extract and process geometry, then populates a Model3D structure with positions, texture coordinates, normals, and vertex colors.

## Core Responsibilities
- Initialize and manage QD3D/Quesa library with lazy initialization and caching
- Load model files from disk and read drawable geometry objects
- Register and operate a custom triangulator renderer to decompose curved surfaces into triangles
- Extract vertex attributes (position, texture coordinates, normal, color) from triangles
- Deduplicate coincident vertices within position and attribute thresholds
- Populate Model3D output structure with deduuplicated geometry and index mappings
- Provide debug output and tessellation configuration hooks

## External Dependencies
- **Quesa** (QD3D successor): Q3Initialize, Q3File_*, Q3Object_*, Q3View_*, Q3Renderer_*, Q3Group_*, Q3Error_*, Q3AttributeSet_*, Q3XObjectHierarchy_*, Q3PixmapDrawContext_New, Q3SubdivisionStyle_Submit
- **Standard C:** ctype.h, stdlib.h, string.h, cstdlib (qsort)
- **C++ STL:** algorithm, vector
- **Project:** cseries.h (macros), QD3D_Loader.h (header), Model3D (output struct), FileHandler.h (FileSpecifier)

# Source_Files/ModelView/QD3D_Loader.h
## File Purpose
Declares the public interface for loading 3D models from QuickDraw 3D / Quesa format files. Provides functions to load models and configure tesselation (surface subdivision) parameters for the loader.

## Core Responsibilities
- Load 3D models from `.3dmf` (QuickDraw 3D / Quesa) format files into `Model3D` objects
- Configure debug output destination for loader diagnostics
- Set tesselation parameters (world-based or constant-factor subdivision) for curved surface handling
- Reset tesselation settings to engine defaults

## External Dependencies
- `stdio.h` ΓÇô provides `FILE` type for debug output
- `Model3D.h` ΓÇô defines the `Model3D` class for storing loaded model data
- `FileHandler.h` ΓÇô defines `FileSpecifier` for cross-platform file I/O abstraction

# Source_Files/ModelView/StudioLoader.cpp
## File Purpose
Binary parser for 3D Studio Max (.3DS) model files. Reads chunk-based binary format hierarchically, extracts vertex positions, texture coordinates, and face indices, and populates a Model3D structure. Supports optional debug output for troubleshooting file format issues.

## Core Responsibilities
- Validate and open .3DS files; verify MASTER chunk identifier
- Parse hierarchical chunk structure (MASTER ΓåÆ EDITOR ΓåÆ OBJECT ΓåÆ TRIMESH ΓåÆ sub-chunks)
- Load vertex positions and texture coordinates from binary streams
- Extract face/polygon data (vertex index triplets)
- Buffer and process raw binary chunk data with little-endian byte order
- Provide optional debug output to file for format validation

## External Dependencies
- **Notable includes:** StudioLoader.h, Packing.h (byte-order macros and StreamToValue functions)
- **External symbols (defined elsewhere):**
  - FileSpecifier, OpenedFile (file abstraction; methods: Open, Read, GetPosition, SetPosition)
  - Model3D (model structure; members: Positions, VertIndices, TxtrCoords, methods: PosBase, VIBase, TCBase, Clear)
  - GLfloat (OpenGL floating-point type)
  - ChunkBuffer container helpers (ChunkBufferBase, ChunkBufferSize, SetChunkBufferSize)

# Source_Files/ModelView/StudioLoader.h
## File Purpose
Declaration-only header for loading 3D Studio Max model files into the Aleph One game engine's internal 3D model format. Provides minimal public API: model loading and debug output configuration.

## Core Responsibilities
- Load `.3ds` (3D Studio Max) files from disk
- Populate a Model3D object with vertex, normal, texture, and bone data
- Configure debug output destination for loader messages

## External Dependencies
- `<stdio.h>` ΓÇö for FILE type (debug output)
- `Model3D.h` ΓÇö defines the Model3D struct and related animation types
- `FileHandler.h` ΓÇö defines FileSpecifier abstraction for cross-platform file I/O
- Implementation module (StudioLoader.cpp or similar) ΓÇö not shown

# Source_Files/ModelView/WavefrontLoader.cpp
## File Purpose
Parses and loads 3D geometry from Wavefront OBJ files into a Model3D structure. Handles vertex deduplication, index format conversion, and polygon triangulation for use by the Aleph One game engine.

## Core Responsibilities
- Parse Wavefront OBJ file format line-by-line with continuation character support
- Extract and store vertex positions, texture coordinates, and surface normals
- Parse face definitions with complex multi-component vertex indexing
- Convert 1-based OBJ indices to 0-based indices; handle negative (from-end) indices
- Deduplicate vertices by comparing index sets; eliminate redundant vertex data
- Triangulate arbitrary polygons into triangle fans
- Validate index ranges and data completeness; report errors via optional debug output

## External Dependencies
- **Classes (defined elsewhere):** `FileSpecifier`, `OpenedFile`, `Model3D`
- **Types:** `GLfloat` (OpenGL type, assumed from context)
- **Headers:** `<ctype.h>`, `<stdlib.h>`, `<string.h>`, `<algorithm>`, `cseries.h`, `WavefrontLoader.h`
- **STL:** `std::vector`, `std::sort`
- **Preprocessor:** Code guarded by `#ifdef HAVE_OPENGL`; Windows headers included if `__WIN32__`

# Source_Files/ModelView/WavefrontLoader.h
## File Purpose
Public interface for loading Alias Wavefront Object (.obj) format 3D models into the engine's `Model3D` structure. Provides model loading and optional debug output routing for diagnostic messages during parse operations.

## Core Responsibilities
- Declare primary model loader entry point for Wavefront files
- Provide debug message routing configuration
- Bridge file I/O (`FileSpecifier`) with 3D model storage (`Model3D`)

## External Dependencies
- `<stdio.h>` ΓÇô FILE type for debug output
- `Model3D.h` ΓÇô `Model3D` struct definition
- `FileHandler.h` ΓÇô `FileSpecifier` class for file I/O abstraction


