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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Model3D_VertexSource` | struct | Vertex with bone indices (two-bone weighting) and blend factor for skeletal animation |
| `Model3D_Bone` | struct | Bone definition with position and tree-traversal flags (Push/Pop for bone-stack traversal) |
| `Model3D_Frame` | struct | Bone transform: offset and rotation angles per bone |
| `Model3D_SeqFrame` | struct | Frame with sequence reference; extends `Model3D_Frame` |
| `Model3D_Transform` | struct | 3├ù4 transformation matrix for space conversion; includes `Identity()` method |
| `Model3D` | struct | Main model container; aggregates all geometry and animation data |

## Global / File-Static State
None.

## Key Functions / Methods

### FindBoundingBox
- **Signature:** `void FindBoundingBox();`
- **Purpose:** Compute axis-aligned bounding box from vertex positions.
- **Inputs:** None (reads internal `Positions`).
- **Outputs/Return:** Updates `BoundingBox[2][3]` (min/max corners).
- **Side effects:** Modifies bounding box state.
- **Calls:** (internal geometry analysis).

### AdjustNormals
- **Signature:** `void AdjustNormals(int NormalType, float SmoothThreshold = 0.5);`
- **Purpose:** Recompute or adjust vertex normals using one of six strategies (None, Original, Reversed, ClockwiseSide, CounterclockwiseSide).
- **Inputs:** `NormalType` enum, optional smoothing threshold.
- **Outputs/Return:** Updates `Normals` array; optionally splits vertices for per-face normals.
- **Side effects:** Modifies `Normals` and potentially `Positions` size.

### FindPositions_Neutral / FindPositions_Frame / FindPositions_Sequence
- **Signature:** 
  - `bool FindPositions_Neutral(bool UseModelTransform);`
  - `bool FindPositions_Frame(bool UseModelTransform, GLshort FrameIndex, GLfloat MixFrac = 0, GLshort AddlFrameIndex = 0);`
  - `bool FindPositions_Sequence(bool UseModelTransform, GLshort SeqIndex, GLshort FrameIndex, GLfloat MixFrac = 0, GLshort AddlFrameIndex = 0);`
- **Purpose:** Compute vertex positions for skeletal animation at rest pose, a specific frame, or a sequence frame.
- **Inputs:** Model transform flag, frame/sequence indices, crossfade fraction for interpolation.
- **Outputs/Return:** Populates `Positions`; returns validity of indices.
- **Side effects:** Computes skinned vertex positions using bones and blending.
- **Notes:** Uses `VtxSources` and bone data; `MixFrac` enables smooth transitions between keyframes.

### BuildInverseVSIndices
- **Signature:** `void BuildInverseVSIndices();`
- **Purpose:** Precompute inverse-index lookup for vertex sources to accelerate position skinning.
- **Inputs:** None (reads `VtxSrcIndices`).
- **Outputs/Return:** Populates `InverseVSIndices` and `InvVSIPointers`.
- **Side effects:** Modifies inverse-index state.

### BuildTrigTables
- **Signature:** `static void BuildTrigTables();`
- **Purpose:** Initialize global trig lookup tables (expects angles in Marathon engine format).
- **Inputs:** None.
- **Outputs/Return:** None (modifies global trig state).
- **Side effects:** Global initialization; should be called once before frame/sequence transforms.

### Helper Accessors
Inline getters returning pointers to array bases: `PosBase()`, `TCBase()`, `NormBase()`, `ColBase()`, `VtxSIBase()`, `VtxSrcBase()`, `NormSrcBase()`, `BoneBase()`, `VIBase()`, `FrameBase()`, `SeqFrmBase()`, `SFPtrBase()`.

## Control Flow Notes
- **Initialization:** Constructor calls `FindBoundingBox()` and sets identity transforms.
- **Model loading:** External code populates `Positions`, `TxtrCoords`, `Normals`, `Colors`, `VertIndices`, and (if animated) `Bones`, `Frames`, `VtxSources`.
- **Animation:** At runtime, call `FindPositions_Frame()` or `FindPositions_Sequence()` to deform vertices for the current animation state.
- **Rendering:** Pass resulting `Positions`, `Normals`, `Colors`, `TxtrCoords` to OpenGL; `VertIndices` defines triangle connectivity.
- **Debug:** `RenderBoundingBox()` optionally draws bounds for culling verification.

## External Dependencies
- **OpenGL:** Platform-conditional includes (`<OpenGL/gl.h>` on macOS, `<GL/gl.h>` elsewhere; Windows-specific `<wingdi.h>` for compatibility).
- **STL:** `<vector>` for all dynamic arrays.
- **Global:** Assumes `BuildTrigTables()` initializes Marathon-format trig lookups elsewhere (e.g., `world.h`).
