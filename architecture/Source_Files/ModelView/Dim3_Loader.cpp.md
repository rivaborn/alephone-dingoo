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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `BoneTagWrapper` | struct | Pair of bone tags: major/minor (vertices) or self/parent (bones) |
| `NameTagWrapper` | struct | Fixed-size string for frame/sequence names |
| `XML_Dim3DataBlock` | class | Root XML parser; manages file buffer and error reporting |
| `XML_BoundingBoxParser` | class | Parses `<Bound_Box>` / `<View_Box>` with size/offset attributes |
| `XML_VertexParser` | class | Parses individual `<v>` vertex with position, normal, bone tags, blend factor |
| `XML_BoneParser` | class | Parses `<Bone>` with position, self-tag, parent-tag |
| `XML_TriVertexParser` | class | Parses triangle vertex reference with vertex ID, UV, normal |
| `XML_FrameParser` | class | Parses `<Pose>` frame container; names frame |
| `XML_FrameBoneParser` | class | Parses bone data within frame: rotation, offset, bone tag |
| `XML_SequenceParser` | class | Parses `<Animation>` sequence container |
| `XML_SeqFrameParser` | class | Parses frame reference in sequence with sway/move overrides |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DBOut` | FILE* | static | Debug output stream (NULL if disabled) |
| `VertexBoneTags` | vector\<BoneTagWrapper\> | static | Per-vertex: major and minor bone tags |
| `BoneOwnTags` | vector\<BoneTagWrapper\> | static | Per-bone: self tag and parent tag |
| `BoneIndices` | vector\<size_t\> | static | Remap table from read order ΓåÆ canonical order |
| `FrameTags` | vector\<NameTagWrapper\> | static | Frame names for sequence frame lookup |
| `ReadFrame` | vector\<Model3D_Frame\> | static | Temporary: bone transform data during frame parse |
| `Normals` | vector\<GLfloat\> | static | Vertex normals (3 floats per source vertex) |
| `ModelPtr` | Model3D* | static | Current model being loaded (set on entry) |
| `XML_DataBlockLoader` | XML_Dim3DataBlock | static | Singleton XML parser instance |
| `Dim3_RootParser`, `Dim3_Parser` | XML_ElementParser | static | Root and Model element parsers |
| `Dim3_ParserInited` | bool | static | Lazy-init flag for parser tree |

## Key Functions / Methods

### LoadModel_Dim3
- **Signature:** `bool LoadModel_Dim3(FileSpecifier& Spec, Model3D& Model, int WhichPass)`
- **Purpose:** Main entry point; opens file, parses XML, post-processes bone hierarchy and normals.
- **Inputs:** FileSpecifier (model file), Model3D (destination), WhichPass (LoadModelDim3_First or LoadModelDim3_Rest)
- **Outputs/Return:** `bool` ΓÇö true if model has positions and vertex indices, false on error
- **Side effects:**
  - Clears model and static vectors on first pass
  - Modifies Model3D with vertices, bones, frames, sequences
  - Reorders bones by hierarchy; updates vertex bone indices
  - Copies normals into Model.NormSources and Model.Normals
  - Calls Model.BuildInverseVSIndices() and Model.FindPositions_Neutral()
- **Calls:** Spec.Open/GetName, OFile.GetLength/Read, Dim3_SetupParseTree, XML_DataBlockLoader.ParseData, Model.BuildInverseVSIndices, Model.FindPositions_Neutral, std::fill, strncmp, assert
- **Notes:** Multi-pass design allows streaming; bone sorting is O(n┬▓) but runs only on first pass; complex tree traversal handles hierarchy with Push/Pop flags

### GetAngle
- **Signature:** `static int16 GetAngle(float InAngle)`
- **Purpose:** Convert degrees to engine's 9-bit angle units (FULL_CIRCLE = 512).
- **Inputs:** InAngle in degrees (can be negative)
- **Outputs/Return:** `int16` normalized to [0, 512)
- **Notes:** Rounds to nearest int; NORMALIZE_ANGLE macro wraps result

### Dim3_SetupParseTree
- **Signature:** `void Dim3_SetupParseTree()`
- **Purpose:** Lazy-init of XML parser hierarchy (called only once).
- **Side effects:** Wires parser parent-child relationships; sets Dim3_ParserInited = true
- **Calls:** XML_ElementParser::AddChild

### Bone Sorting / Hierarchy Traversal (within LoadModel_Dim3)
Recursively orders bones by depth-first tree traversal of parent-child links:
- Scans BoneOwnTags to find unlisted bones whose parent is on the stack
- Sets Model3D_Bone::Push/Pop flags for stack push/pop boundaries
- Remaps VtxSources.Bone0/Bone1 indices using BoneIndices lookup
- **Notes:** Asserts on circular bone references; stack-based to avoid recursion

### XML_VertexParser / XML_BoneParser / XML_FrameBoneParser / etc.
- **Purpose:** Each class implements one XML element type
- **Pattern:** Start() initializes local state, HandleAttribute() parses tag=value pairs, AttributesDone() commits to model/temp storage
- **Calls:** Parsers call ModelΓåÆpush_back or vectorΓåÆpush_back; use strncpy for tag storage, sscanf/ReadFloatValue/ReadUInt16Value for attribute parsing

## Control Flow Notes
**Loading Pipeline:**
1. **File I/O:** LoadModel_Dim3 reads entire file into memory
2. **XML Parse:** XML_Dim3DataBlock parses XML, routes to element-specific parsers
3. **Element Parsing:** Vertices, Bones, Frames, Sequences are accumulated in static vectors
4. **Post-Process:** Bone hierarchy is topologically sorted; normals are remapped; model is finalized
5. **Result:** Model3D is ready for use in render/animation pipeline

Model loading is strictly initialization-phase; no frame-by-frame updates here.

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
