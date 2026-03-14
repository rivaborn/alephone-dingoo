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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ChunkHeaderData | struct | Container for chunk ID (uint16) and byte size (uint32); constant size 6 bytes |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| DBOut | FILE* | static | Debug output destination; NULL if not set |
| ChunkBuffer | vector<uint8> | static | Reusable byte buffer for loading chunk contents |
| ModelPtr | Model3D* | static | Pointer to the model being populated; set by LoadModel_Studio |

## Key Functions / Methods

### LoadModel_Studio
- **Signature:** `bool LoadModel_Studio(FileSpecifier& Spec, Model3D& Model)`
- **Purpose:** Entry point; opens a .3DS file and populates a Model3D object
- **Inputs:** FileSpecifier (file path), Model3D reference (to populate)
- **Outputs/Return:** `true` if load succeeds and model has positions and vertex indices; `false` on error
- **Side effects:** Sets ModelPtr global; clears Model; may write to DBOut
- **Calls:** Spec.Open(), ReadChunkHeader(), ReadContainer()
- **Notes:** Validates that first chunk is MASTER (0x4d4d); exits early if file is invalid

### ReadChunkHeader
- **Signature:** `bool ReadChunkHeader(OpenedFile& OFile, ChunkHeaderData& ChunkHeader)`
- **Purpose:** Deserialize a 6-byte chunk header from file stream
- **Inputs:** OpenedFile, output ChunkHeaderData struct
- **Outputs/Return:** `true` on success; `false` if read fails
- **Side effects:** Advances file position; logs error to DBOut on failure
- **Calls:** OFile.Read(), StreamToValue()
- **Notes:** Uses little-endian Packing.h routines

### LoadChunk
- **Signature:** `bool LoadChunk(OpenedFile& OFile, ChunkHeaderData& ChunkHeader)`
- **Purpose:** Read entire chunk data (excluding header) into ChunkBuffer
- **Inputs:** OpenedFile, ChunkHeaderData with size
- **Outputs/Return:** `true` on success
- **Side effects:** Resizes ChunkBuffer; advances file position; logs to DBOut
- **Calls:** OFile.Read(), SetChunkBufferSize()
- **Notes:** Subtracts header size (6 bytes) from chunk size

### SkipChunk
- **Signature:** `bool SkipChunk(OpenedFile& OFile, ChunkHeaderData& ChunkHeader)`
- **Purpose:** Seek past a chunk without reading its contents
- **Inputs:** OpenedFile, ChunkHeaderData with size
- **Outputs/Return:** `true` on success
- **Side effects:** Seeks file position forward; logs to DBOut
- **Calls:** OFile.GetPosition(), OFile.SetPosition()

### ReadContainer
- **Signature:** `bool ReadContainer(OpenedFile& OFile, ChunkHeaderData& ChunkHeader, bool (*ContainerCallback)(OpenedFile&, int32))`
- **Purpose:** Generic container reader; computes chunk boundary and invokes callback
- **Inputs:** OpenedFile, ChunkHeaderData, callback function pointer
- **Outputs/Return:** `true` if callback succeeds
- **Side effects:** Logs container entry/exit to DBOut
- **Calls:** OFile.GetPosition(), ContainerCallback
- **Notes:** Callback is passed end-of-chunk file position; no additional validation

### ReadMaster, ReadEditor, ReadObject, ReadTrimesh, ReadFaceData
- **Signature:** `bool Read<Type>(OpenedFile& OFile, int32 ParentChunkEnd)` (all similar pattern)
- **Purpose:** Parse specific chunk types; loop through child chunks until parent boundary, route via switch statement
- **Inputs:** OpenedFile, parent chunk end position
- **Outputs/Return:** `true` on success; `false` if position overruns boundary or read fails
- **Side effects:** ReadObject logs object name; ReadTrimesh/ReadFaceData populate ModelPtr; all log errors
- **Calls:** ReadChunkHeader(), switch on chunk ID, LoadChunk()/SkipChunk()/ReadContainer()
- **Notes:** ReadFaceData directly populates ModelPtrΓåÆVertIndices; all validate position Γëñ ParentChunkEnd

### LoadVertices
- **Signature:** `static void LoadVertices()`
- **Purpose:** Extract vertex position data from ChunkBuffer and populate ModelPtrΓåÆPositions
- **Inputs:** None (uses global ChunkBuffer)
- **Outputs/Return:** None
- **Side effects:** Resizes ModelPtrΓåÆPositions; calls LoadFloats
- **Calls:** ChunkBufferBase(), StreamToValue(), LoadFloats()
- **Notes:** First uint16 is vertex count; data is 3 floats per vertex

### LoadTextureCoordinates
- **Signature:** `static void LoadTextureCoordinates()`
- **Purpose:** Extract texture coordinate data from ChunkBuffer and populate ModelPtrΓåÆTxtrCoords
- **Inputs:** None (uses global ChunkBuffer)
- **Outputs/Return:** None
- **Side effects:** Resizes ModelPtrΓåÆTxtrCoords; calls LoadFloats()
- **Calls:** ChunkBufferBase(), StreamToValue(), LoadFloats()
- **Notes:** First uint16 is coordinate count; data is 2 floats per coordinate

### LoadFloats
- **Signature:** `void LoadFloats(int NVals, uint8 *Stream, GLfloat *Floats)`
- **Purpose:** Convert binary stream of IEEE 754 4-byte integers to GLfloat array via byte-copy
- **Inputs:** NVals (count), Stream (source bytes), Floats (output array)
- **Outputs/Return:** None; writes to Floats
- **Side effects:** Populates Floats array; advances Stream pointer
- **Calls:** StreamToValue()
- **Notes:** Uses byte-copy to preserve IEEE 754 bit representation; assumes sizeof(GLfloat) == 4 (asserted)

## Control Flow Notes
- **Init/Load sequence:** LoadModel_Studio() validates file, opens, calls ReadContainer(ReadMaster)
- **Recursive parsing:** ReadMaster ΓåÆ ReadEditor ΓåÆ ReadObject ΓåÆ ReadTrimesh, each looping and dispatching to child handlers
- **Data extraction:** Vertex/texture chunks buffered in ChunkBuffer, then processed in-place; face data read directly into model
- **Shutdown:** Implicit; no explicit cleanup (model remains in caller's scope)

## External Dependencies
- **Notable includes:** StudioLoader.h, Packing.h (byte-order macros and StreamToValue functions)
- **External symbols (defined elsewhere):**
  - FileSpecifier, OpenedFile (file abstraction; methods: Open, Read, GetPosition, SetPosition)
  - Model3D (model structure; members: Positions, VertIndices, TxtrCoords, methods: PosBase, VIBase, TCBase, Clear)
  - GLfloat (OpenGL floating-point type)
  - ChunkBuffer container helpers (ChunkBufferBase, ChunkBufferSize, SetChunkBufferSize)
