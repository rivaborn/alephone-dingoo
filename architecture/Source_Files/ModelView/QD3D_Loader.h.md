# Source_Files/ModelView/QD3D_Loader.h

## File Purpose
Declares the public interface for loading 3D models from QuickDraw 3D / Quesa format files. Provides functions to load models and configure tesselation (surface subdivision) parameters for the loader.

## Core Responsibilities
- Load 3D models from `.3dmf` (QuickDraw 3D / Quesa) format files into `Model3D` objects
- Configure debug output destination for loader diagnostics
- Set tesselation parameters (world-based or constant-factor subdivision) for curved surface handling
- Reset tesselation settings to engine defaults

## Key Types / Data Structures
None defined in this file.

| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class (external) | File path abstraction for cross-platform file I/O |
| `Model3D` | struct (external) | 3D model container with vertices, normals, bones, animation frames |

## Global / File-Static State
None.

## Key Functions / Methods

### LoadModel_QD3D
- **Signature:** `bool LoadModel_QD3D(FileSpecifier& Spec, Model3D& Model);`
- **Purpose:** Load a 3D model from a QuickDraw 3D / Quesa format file.
- **Inputs:** `Spec` ΓÇô file specification pointing to the model file; `Model` ΓÇô destination Model3D object (filled by the function)
- **Outputs/Return:** `bool` ΓÇô success/failure
- **Side effects:** Populates the Model3D object with geometry, normals, bones, and animation data; may allocate memory within Model3D
- **Calls (direct):** None visible in header (implementation in `.cpp`)
- **Notes:** Tesselation is applied based on current settings (see `SetTesselation_QD3D`); coordinate system transformation may be applied

### SetDebugOutput_QD3D
- **Signature:** `void SetDebugOutput_QD3D(FILE *DebugOutput);`
- **Purpose:** Configure where the loader emits status and diagnostic messages.
- **Inputs:** `DebugOutput` ΓÇô FILE stream pointer (e.g., `stdout`, `stderr`, or a log file)
- **Outputs/Return:** None
- **Side effects:** Sets a global or static FILE pointer used by the loader
- **Notes:** Passing `NULL` likely disables debug output

### SetTesselation_QD3D
- **Signature:** `void SetTesselation_QD3D(bool IsWorldLength, float TessLength);`
- **Purpose:** Configure how curved surfaces are subdivided into triangles during loading.
- **Inputs:** 
  - `IsWorldLength` ΓÇô if `true`, `TessLength` is in world-geometry units; if `false`, it is a constant subdivision factor
  - `TessLength` ΓÇô the tesselation parameter (scale or factor)
- **Outputs/Return:** None
- **Side effects:** Sets global/static tesselation configuration used by subsequent `LoadModel_QD3D` calls
- **Notes:** Higher tesselation (finer subdivision) increases triangle count and rendering cost; applies to NURBS and other curved primitives

### SetDefaultTesselation_QD3D
- **Signature:** `void SetDefaultTesselation_QD3D();`
- **Purpose:** Reset tesselation parameters to engine defaults.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Overwrites current tesselation settings

## Control Flow Notes
This header is part of the model-loading subsystem, likely called during engine initialization or level-load phases. The `SetTesselation_QD3D` and `SetDebugOutput_QD3D` functions configure global state before any models are loaded. `LoadModel_QD3D` is the primary entry point for importing `.3dmf` files into the `Model3D` data structure for rendering and animation.

## External Dependencies
- `stdio.h` ΓÇô provides `FILE` type for debug output
- `Model3D.h` ΓÇô defines the `Model3D` class for storing loaded model data
- `FileHandler.h` ΓÇô defines `FileSpecifier` for cross-platform file I/O abstraction
