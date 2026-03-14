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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| FullVertexData | struct | Consolidated vertex data with interleaved GLfloat components: position (3), texcoord (2), normal (3), color (3) |
| Model3D | struct (external) | Output geometry container with arrays and per-vertex remapping indices |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| DBOut | FILE* | static | Debug output stream (optional) |
| QD3D_Present | bool | static | Cached flag: QD3D/Quesa available |
| QD3D_Presence_Checked | bool | static | Cached flag: initialization was attempted |
| TesselationData | TQ3SubdivisionStyleData | static | Tessellation method and parameters for curved surfaces |
| TriangulatorClass | TQ3XObjectClass | static | Registered custom renderer class |
| TriangulatorClassType | TQ3ObjectType | static | Type ID of triangulator renderer |
| TriangulatorView | TQ3ViewObject | static | Cached fake view/renderer for geometry extraction |
| TxtrCoordsPresent | bool | static | Whether current model has texture coordinates (set per-triangle) |
| NormalsPresent | bool | static | Whether current model has vertex normals (set per-triangle) |
| ColorsPresent | bool | static | Whether current model has vertex colors (set per-triangle) |
| FullVertexList | vector<FullVertexData> | static | Accumulated triangle vertices during tessellation pass |
| Thresholds | GLfloat[11] | static | Component-wise comparison thresholds for vertex deduplication |

## Key Functions / Methods

### LoadModel_QD3D
- **Signature:** `bool LoadModel_QD3D(FileSpecifier& Spec, Model3D& Model)`
- **Purpose:** Main entry point. Initializes QD3D, loads a model file, tessellates into triangles, deduplicates vertices, and populates Model3D.
- **Inputs:** FileSpecifier (file path/descriptor), Model3D reference (output)
- **Outputs/Return:** bool (success); Model populated on true
- **Side effects:** 
  - Lazy-initializes QD3D and registers error handler (one-time)
  - Clears and fills Model
  - May print debug messages via DBOut
  - Uses static TriangulatorView for tessellation
- **Calls:** Q3Initialize, Q3Error_Register, CreateTriangulator, LoadModel, StartAccumulatingVertices, Q3View_StartRendering, Q3SubdivisionStyle_Submit, Q3Object_Submit, Q3View_EndRendering, Q3Object_Dispose, GetVerticesIntoModel, Q3Exit
- **Notes:** Initialization is one-shot and cached. Tessellation loop may retraverse if Q3View_EndRendering returns kQ3ViewStatusRetraverse.

### LoadModel
- **Signature:** `TQ3Object LoadModel(FileSpecifier& Spec)`
- **Purpose:** Opens a QD3D file, reads all drawable objects, and groups them.
- **Inputs:** FileSpecifier (file path)
- **Outputs/Return:** TQ3Object (display group, or NULL on error)
- **Side effects:** File I/O; allocates QD3D objects
- **Calls:** Q3FSSpecStorage_New, Q3File_New, Q3File_SetStorage, Q3File_OpenRead, Q3File_IsEndOfFile, Q3DisplayGroup_New, Q3File_ReadObject, Q3Object_IsDrawable, Q3Group_AddObject, Q3Object_Dispose, Q3Error_Get
- **Notes:** Filters to drawable objects only (skips hints, comments). Returns NULL if any step fails.

### CreateTriangulator
- **Signature:** `bool CreateTriangulator(void)`
- **Purpose:** Registers and caches a custom renderer that intercepts geometry submissions and triangulates all surfaces.
- **Inputs:** None
- **Outputs/Return:** bool (success)
- **Side effects:** 
  - Registers custom renderer class (TriangulatorClass, TriangulatorClassType)
  - Creates and caches TriangulatorView with a dummy pixmap draw context
- **Calls:** Q3XObjectHierarchy_RegisterClass, Q3PixmapDrawContext_New, Q3Renderer_NewFromType, Q3View_New, Q3View_SetDrawContext, Q3View_SetRenderer, Q3Object_Dispose
- **Notes:** Idempotent; early-returns if TriangulatorView already exists. Dummy context has zero size (geometry extraction only, no rendering).

### TriangulatorGeometry_Triangle
- **Signature:** `static TQ3Status TriangulatorGeometry_Triangle(TQ3ViewObject View, void *PrivateData, TQ3GeometryObject Triangle, const TQ3TriangleData *TriangleData)`
- **Purpose:** Geometry sink callback. Extracts vertex data from each triangle submitted to the renderer.
- **Inputs:** TriangleData (3 vertices with optional attributes: position, texcoord, normal, color)
- **Outputs/Return:** kQ3Success
- **Side effects:** 
  - Appends 3 FullVertexData entries to FullVertexList
  - Updates TxtrCoordsPresent, NormalsPresent, ColorsPresent (set to false if attribute missing in any vertex)
- **Calls:** Q3AttributeSet_Contains, Q3AttributeSet_Get
- **Notes:** Looks for ShadingUV or SurfaceUV for texture coordinates. Sets "Present" flags to false if an attribute is absent, which suppresses that attribute in the final model.

### GetVerticesIntoModel
- **Signature:** `void GetVerticesIntoModel(Model3D& Model)`
- **Purpose:** Post-processes accumulated vertices: deduplicates via lexicographic comparison, builds index remapping, populates Model3D arrays.
- **Inputs:** FullVertexList, Thresholds (global state)
- **Outputs/Return:** Model (filled: Positions, TxtrCoords, Normals, Colors, VertIndices)
- **Side effects:** 
  - Resizes and populates Model arrays (conditionally, based on "Present" flags)
  - Clears FullVertexList
- **Calls:** qsort, CompareVertices, MIN, MAX
- **Notes:** Deduplication uses position-relative thresholds for position, and fixed small thresholds for other attributes. Original indices map to deduplicated vertex ranks via Model.VertIndices.

### CompareVertices
- **Signature:** `int CompareVertices(const void *VI1, const void *VI2)`
- **Purpose:** qsort comparator. Lexicographically compares two vertices (by index) using component thresholds.
- **Inputs:** Two int pointers (indices into FullVertexList)
- **Outputs/Return:** -1 (first < second), 0 (within thresholds), or 1 (first > second)
- **Side effects:** Reads FullVertexList and Thresholds (global)
- **Notes:** Reversed return (-1 for greater, 1 for lesser) creates descending sort order. Returns 0 if all components are within respective thresholds; this groups duplicate vertices.

**Trivial helpers (SetDebugOutput_QD3D, SetTesselation_QD3D, SetDefaultTesselation_QD3D, QD3D_Error_Handler, TriangulatorMetaHandler, TriangulatorGeometry_MetaHandler, TriangulatorStartFrame, TriangulatorStartPass, TriangulatorEndPass, TriangulatorCancel, StartAccumulatingVertices):** Configuration setters, lifecycle stubs, and dispatcher methods.

## Control Flow Notes
1. **Init (once per session):** LoadModel_QD3D lazy-initializes Q3D, registers error handler, creates triangulator
2. **Load & Tessellate:**
   - LoadModel reads file and builds object hierarchy
   - StartAccumulatingVertices clears state
   - Q3View_StartRendering begins "rendering" pass
   - Q3Object_Submit sends model; TriangulatorGeometry_Triangle called per triangle
   - Q3View_EndRendering may loop if retraversal requested
3. **Dedup & Output:** GetVerticesIntoModel sorts, groups, and builds final Model3D

## External Dependencies
- **Quesa** (QD3D successor): Q3Initialize, Q3File_*, Q3Object_*, Q3View_*, Q3Renderer_*, Q3Group_*, Q3Error_*, Q3AttributeSet_*, Q3XObjectHierarchy_*, Q3PixmapDrawContext_New, Q3SubdivisionStyle_Submit
- **Standard C:** ctype.h, stdlib.h, string.h, cstdlib (qsort)
- **C++ STL:** algorithm, vector
- **Project:** cseries.h (macros), QD3D_Loader.h (header), Model3D (output struct), FileHandler.h (FileSpecifier)
