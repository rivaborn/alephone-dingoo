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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Model3D_Transform` | struct | 3├ù4 matrix for affine 3D transformations (rotation + translation) |
| `Model3D_VertexSource` | struct | Vertex position and dual-bone blend weights; defines animation source |
| `Model3D_Bone` | struct | Bone position and tree traversal flags (push/pop) |
| `Model3D_Frame` | struct | Bone transform: offset (3D) and angles (3 rotations) |
| `Model3D_SeqFrame` | struct | Frame plus sequence frame index for sequence-based animation |
| `FlaggedVector` | struct (local) | 3D vector + validity flag for per-polygon/vertex normal computation |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `BoneMatrices` | `vector<Model3D_Transform>` | static | Scratch array for bone transformation matrices during frame computation |
| `BoneStack` | `vector<size_t>` | static | Scratch stack for bone hierarchy traversal (push/pop semantics) |
| `TrigNorm` | `const GLfloat` | file-static | Normalization factor for cosine/sine table lookups |

## Key Functions / Methods

### TransformPoint
- **Signature:** `void TransformPoint(GLfloat *Dest, GLfloat *Src, Model3D_Transform& T)`
- **Purpose:** Apply 3D affine transformation (rotation + translation) to a point.
- **Inputs:** source point array (3 floats), transformation matrix
- **Outputs/Return:** destination point array (3 floats, modified in place)
- **Side effects:** Writes to Dest; does not modify Src or T
- **Calls:** `ScalarProd()` (dot product)
- **Notes:** Src and Dest must be different arrays; implements `Dest = T.M ├ù Src + T.M[*][3]`

### TransformVector
- **Signature:** `void TransformVector(GLfloat *Dest, GLfloat *Src, Model3D_Transform& T)`
- **Purpose:** Apply rotation (no translation) to a vector; used for normals.
- **Inputs:** source vector (3 floats), transformation matrix
- **Outputs/Return:** destination vector (3 floats)
- **Side effects:** Writes to Dest
- **Calls:** `ScalarProd()`
- **Notes:** Omits translation component; `Src` and `Dest` must differ

### FindFrameTransform
- **Signature:** `void FindFrameTransform(Model3D_Transform& T, Model3D_Frame& Frame, GLfloat MixFrac, Model3D_Frame& AddlFrame)`
- **Purpose:** Build transformation matrix from frame data (three rotation angles + offset), with optional interpolation between two frames.
- **Inputs:** primary frame, mix fraction (0ΓÇô1 for blending), secondary frame for crossfade
- **Outputs/Return:** transformation matrix T
- **Side effects:** Modifies T; reads cosine_table, sine_table
- **Calls:** `InterpolateAngle()`, cosine/sine table lookups, matrix rotation operations
- **Notes:** Applies rotations in ZΓåÆXΓåÆY order (Marathon/Tomb Raider convention); interpolates angles and offsets

### FindBoneTransform
- **Signature:** `void FindBoneTransform(Model3D_Transform& T, Model3D_Bone& Bone, Model3D_Frame& Frame, GLfloat MixFrac, Model3D_Frame& AddlFrame)`
- **Purpose:** Build bone's local transformation by combining frame transform with bone's base position.
- **Inputs:** bone (position), frame, mix fraction, additional frame
- **Outputs/Return:** transformation matrix T
- **Side effects:** Modifies T
- **Calls:** `FindFrameTransform()`, `ScalarProd()`
- **Notes:** Adjusts translation to account for bone's offset from origin

### TMatMultiply
- **Signature:** `void TMatMultiply(Model3D_Transform& Res, Model3D_Transform& A, Model3D_Transform& B)`
- **Purpose:** Matrix multiplication: Res = A ├ù B.
- **Inputs:** matrices A and B
- **Outputs/Return:** result matrix Res
- **Side effects:** Modifies Res
- **Calls:** None (inline computation)
- **Notes:** Separates 3├ù3 rotation and translation components for clarity

### AdjustNormals
- **Signature:** `void Model3D::AdjustNormals(int NormalType, float SmoothThreshold)`
- **Purpose:** Process normals: normalize, reverse direction, or recompute per-polygon/vertex with optional smooth-shading.
- **Inputs:** NormalType (None, Original, Reversed, ClockwiseSide, CounterclockwiseSide), smooth threshold (variance tolerance)
- **Outputs/Return:** None
- **Side effects:** Resizes/rewrites Positions, Normals, TxtrCoords, Colors, VtxSrcIndices (may increase vertex count); reads from NormSources, VtxSources
- **Calls:** `NormalizeNormal()`, cross product, variance computation, vertex splitting
- **Notes:** Complex algorithm for ClockwiseSide/CounterclockwiseSide: computes per-polygon normals, calculates per-vertex variance, splits vertices at hard edges exceeding threshold

### FindPositions_Neutral
- **Signature:** `bool Model3D::FindPositions_Neutral(bool UseModelTransform)`
- **Purpose:** Copy vertex positions from sources (no animation/skeletal deformation).
- **Inputs:** UseModelTransform flag (apply model's overall transform)
- **Outputs/Return:** true if vertex sources were used; false if no sources
- **Side effects:** Resizes and populates Positions, Normals; may apply TransformPos/TransformNorm
- **Calls:** `TransformPoint()`, `TransformVector()`, `objlist_copy()`
- **Notes:** Returns false for static meshes without vertex sources

### FindPositions_Frame
- **Signature:** `bool Model3D::FindPositions_Frame(bool UseModelTransform, GLshort FrameIndex, GLfloat MixFrac = 0, GLshort AddlFrameIndex = 0)`
- **Purpose:** Compute vertex positions using skeletal animation: apply bone transforms to vertex sources.
- **Inputs:** frame index, mix fraction (crossfade between two frames), secondary frame index, UseModelTransform flag
- **Outputs/Return:** true if successful; false on out-of-range indices
- **Side effects:** Resizes Positions, Normals; uses/resizes BoneMatrices, BoneStack; calls BuildInverseVSIndices if needed
- **Calls:** `FindBoneTransform()`, `TMatMultiply()`, `TransformPoint()`, `TransformVector()`, bone hierarchy traversal
- **Notes:** Handles bone tree with push/pop stack; supports dual-bone per-vertex blending; optional final model transform

### FindPositions_Sequence
- **Signature:** `bool Model3D::FindPositions_Sequence(bool UseModelTransform, GLshort SeqIndex, GLshort FrameIndex, GLfloat MixFrac = 0, GLshort AddlFrameIndex = 0)`
- **Purpose:** Compute vertex positions using sequence-based animation (extends frame animation with per-sequence transforms).
- **Inputs:** sequence index, frame index within sequence, mix fraction, secondary frame index, UseModelTransform flag
- **Outputs/Return:** true if successful; false on out-of-range indices
- **Side effects:** Resizes Positions, Normals; calls FindPositions_Frame
- **Calls:** `FindFrameTransform()`, `FindPositions_Frame()`, `TMatMultiply()`, `TransformPoint()`, `TransformVector()`
- **Notes:** Sequence provides additional transformation applied after frame-based deformation

### Clear
- **Signature:** `void Model3D::Clear()`
- **Purpose:** Erase all model data.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears all member vectors (Positions, Normals, Colors, Bones, Frames, etc.); recalculates bounding box
- **Calls:** `clear()` on all vectors, `FindBoundingBox()`
- **Notes:** Destructive operation

### RenderBoundingBox
- **Signature:** `void Model3D::RenderBoundingBox(const GLfloat *EdgeColor, const GLfloat *DiagonalColor)`
- **Purpose:** Debug rendering of bounding box edges and face diagonals.
- **Inputs:** optional RGB colors for edges and diagonals (NULL to skip)
- **Outputs/Return:** None
- **Side effects:** Issues OpenGL draw calls; modifies GL state (disables textures, enables/disables vertex arrays)
- **Calls:** `glDisable()`, `glEnableClientState()`, `glVertexPointer()`, `glDrawElements()`, `SglColor3fv()`
- **Notes:** Skips rendering sets with NULL color pointers

### BuildInverseVSIndices
- **Signature:** `void Model3D::BuildInverseVSIndices()`
- **Purpose:** Build inverse mapping from vertex sources to vertex indices for efficient animation lookup.
- **Inputs:** VtxSrcIndices (maps each vertex to its source)
- **Outputs/Return:** None
- **Side effects:** Populates InverseVSIndices, InvVSIPointers (cumulative index array)
- **Calls:** `objlist_clear()`
- **Notes:** InvVSIPointers has one extra element (pointer just past end); called lazily by FindPositions_Frame

## Control Flow Notes

**Initialization:**
- Constructor initializes bounding box and identity transforms.
- `BuildTrigTables()` should be called before frame-based animation (may be called elsewhere in engine).
- `BuildInverseVSIndices()` called lazily by `FindPositions_Frame()`.

**Animation pipeline:**
- Neutral mode: Direct copy from vertex sources (static or in reference frame).
- Frame mode: Compute bone matrices from frames, apply to vertex sources, blend dual-bone influences.
- Sequence mode: Call frame mode, then apply sequence-level transform.
- Optional final model transform applied if `UseModelTransform = true`.

**Normal adjustment:**
- `AdjustNormals()` handles five modes: erase normals, use original, reverse, or recompute per-polygon with optional vertex splitting for smooth shading.
- Variance-based threshold decides whether adjacent polygons should share a vertex normal or force vertex split.

## External Dependencies

- **VecOps.h:** Vector operation templates (`VecCopy`, `VecSub`, `ScalarProd`, `VecAddTo`, `VecScalarMultTo`)
- **cseries.h:** Compatibility layer (`objlist_copy`, `objlist_clear` macros; cross-platform type definitions)
- **world.h:** Engine math (`angle`, `cosine_table`, `sine_table`, `TRIG_MAGNITUDE`, `build_trig_tables()`, `NORMALIZE_ANGLE`)
- **Model3D.h:** Data structure definitions (`Model3D`, `Model3D_Transform`, `Model3D_Bone`, `Model3D_Frame`, etc.)
- **OGL_Setup.h:** OpenGL utilities (`SglColor3fv()` color setter)
- **GL/gl.h:** OpenGL C API (`glDisable`, `glEnableClientState`, `glVertexPointer`, `glDrawElements`)
- **Defined elsewhere:** `UNONE` (sentinel value, likely max unsigned), `TEST_FLAG`, `MAX`, `MIN`, `sqrt` (math library)
