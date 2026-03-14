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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `IndexedVertListCompare` | struct | STL comparator for sorting vertex index sets; enables deduplication by finding unique combinations of position/texture/normal indices |
| `Present_Position`, `Present_TxtrCoord`, `Present_Normal` | enum constants | Bit flags indicating which vertex components are present across all vertices |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DBOut` | `FILE*` | static | Debug output file handle; if non-null, receives status and error messages |
| `InputLine` | `vector<char>` | static | Dynamically-sized buffer for reading OBJ lines; supports line continuation with `\` |

## Key Functions / Methods

### SetDebugOutput_Wavefront
- **Signature:** `void SetDebugOutput_Wavefront(FILE *DebugOutput)`
- **Purpose:** Configure the destination for debug and error messages.
- **Inputs:** `DebugOutput` ΓÇö file pointer (may be NULL to disable output).
- **Outputs/Return:** None.
- **Side effects:** Modifies static global `DBOut`.
- **Calls:** (none)
- **Notes:** No-op if passed NULL.

### LoadModel_Wavefront
- **Signature:** `bool LoadModel_Wavefront(FileSpecifier& Spec, Model3D& Model)`
- **Purpose:** Main loader function; reads a Wavefront OBJ file and populates a Model3D with deduplicated geometry.
- **Inputs:** `Spec` ΓÇö file reference to OBJ file; `Model` ΓÇö target geometry container (cleared on entry).
- **Outputs/Return:** `true` on success; `false` if file open fails, no valid geometry found, or index validation fails. Populates `Model.Positions`, `Model.TxtrCoords`, `Model.Normals`, `Model.VertIndices`.
- **Side effects:** Clears `Model`; allocates temporary vectors; reads file I/O; writes to `DBOut` if configured. Converts all polygons to triangle fans and stores in `Model.VertIndices`.
- **Calls:** `Model.Clear()`, `Spec.GetName()`, `Spec.Open()`, `OFile.Read()`, `CompareToKeyword()`, `GetVertIndxSet()`, `sscanf()`, `std::sort()`, vector push_back methods.
- **Notes:** 
  - Handles OBJ line continuations (`\` at line end).
  - Converts 1-based OBJ indices to 0-based; negative indices resolve from list end.
  - Deduplicates vertices by sorting index sets via `IndexedVertListCompare` and comparing `(PosIndx, TCIndx, NormIndx)` tuples.
  - Validates that all vertices have position indices; optionally checks texture and normal indices if present in all vertices.
  - Fan-triangulates all polygons (first vertex + each subsequent edge pair).
  - Requires at least 3 vertices per polygon (warns and skips smaller).

### CompareToKeyword
- **Signature:** `char *CompareToKeyword(const char *Keyword)`
- **Purpose:** Match the beginning of `InputLine` against a keyword; return pointer to remainder if matched.
- **Inputs:** `Keyword` ΓÇö C string to match against line prefix.
- **Outputs/Return:** Pointer to first non-whitespace character after keyword if matched; NULL if no match or keyword ends line.
- **Side effects:** None (examines static `InputLine` only).
- **Calls:** `strlen()`.
- **Notes:** Requires exact match followed by whitespace or end-of-line. Returns NULL if non-whitespace immediately follows keyword (e.g., "vex" does not match "v").

### GetVertIndxSet
- **Signature:** `char *GetVertIndxSet(char *Buffer, short& Presence, short& PosIndx, short& TCIndx, short& NormIndx)`
- **Purpose:** Parse one vertex index set from a face line (e.g., "1/2/3" or "1//3").
- **Inputs:** `Buffer` ΓÇö pointer into face data; output parameters passed by reference.
- **Outputs/Return:** Pointer to next unparsed character; populates `Presence` (bitmask of Present_* flags), and `PosIndx`, `TCIndx`, `NormIndx`.
- **Side effects:** None (Buffer modified by value, not reference).
- **Calls:** `GetVertIndx()` (three times, once per index component).
- **Notes:** Skips leading whitespace; returns NULL at end-of-string. Sets Presence flags only for indices actually found; missing components (e.g., "1//3" has no texture coordinate) remain zero.

### GetVertIndx
- **Signature:** `char *GetVertIndx(char *Buffer, bool& WasFound, short& Val, bool& HitEnd)`
- **Purpose:** Parse a single vertex index value; advance past delimiters.
- **Inputs:** `Buffer` ΓÇö pointer into index data; output parameters passed by reference.
- **Outputs/Return:** Pointer after index (past '/', space, tab, or null); sets `WasFound` (true if sscanf succeeded), `Val` (parsed index), `HitEnd` (true if space/tab/null hit, indicating end of vertex set).
- **Side effects:** None.
- **Calls:** `sscanf()` with format `"%hd"`.
- **Notes:** Stops at '/' (separates position/texture/normal indices), whitespace, or null. Does not validate index range here.

---

## Control Flow Notes
**Initialization ΓåÆ Parsing ΓåÆ Validation ΓåÆ Deduplication ΓåÆ Output:**

1. **Init:** Clear Model; open file.
2. **Parse:** Read lines with continuation support (`\`). Skip comments. Dispatch on keyword:
   - `v`, `vt`, `vn`: Append float data to temporary vectors.
   - `f`: Parse vertex index sets; accumulate indices and polygon size.
3. **Validate:** Check all vertices have position indices; ensure indices are in range.
4. **Deduplicate:** Sort vertex index sets; scan in order; assign unique IDs based on `(pos, texcoord, normal)` tuples.
5. **Output:** Build final vertex lists and triangle index list; fan-triangulate all polygons.

Not inferable from this file: how Model3D is initialized, what data members it contains, or what happens downstream with the loaded geometry.

## External Dependencies
- **Classes (defined elsewhere):** `FileSpecifier`, `OpenedFile`, `Model3D`
- **Types:** `GLfloat` (OpenGL type, assumed from context)
- **Headers:** `<ctype.h>`, `<stdlib.h>`, `<string.h>`, `<algorithm>`, `cseries.h`, `WavefrontLoader.h`
- **STL:** `std::vector`, `std::sort`
- **Preprocessor:** Code guarded by `#ifdef HAVE_OPENGL`; Windows headers included if `__WIN32__`
