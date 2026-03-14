# Source_Files/ModelView/StudioLoader.h

## File Purpose
Declaration-only header for loading 3D Studio Max model files into the Aleph One game engine's internal 3D model format. Provides minimal public API: model loading and debug output configuration.

## Core Responsibilities
- Load `.3ds` (3D Studio Max) files from disk
- Populate a Model3D object with vertex, normal, texture, and bone data
- Configure debug output destination for loader messages

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FileSpecifier | class (defined in FileHandler.h) | Represents a file path; abstraction over platform-specific file I/O |
| Model3D | struct (defined in Model3D.h) | OpenGL-friendly 3D model container with positions, normals, textures, bones, frames, and animation sequences |

## Global / File-Static State
None.

## Key Functions / Methods

### LoadModel_Studio
- **Signature:** `bool LoadModel_Studio(FileSpecifier& Spec, Model3D& Model)`
- **Purpose:** Load a 3D Studio Max model file from disk and populate a Model3D object
- **Inputs:** 
  - `FileSpecifier& Spec`: File path to the `.3ds` model file
  - `Model3D& Model`: Output model structure (populated on success)
- **Outputs/Return:** `bool` ΓÇö true if load succeeded, false otherwise
- **Side effects:** Reads from disk; modifies the Model3D object's internal vectors (Positions, Normals, TxtrCoords, etc.)
- **Calls:** Implementation not visible in this header (defined elsewhere, likely StudioLoader.cpp)
- **Notes:** Signature indicates it works with both boned (skeletal) and non-boned models. Error handling appears to be via return value only (no exception throwing).

### SetDebugOutput_Studio
- **Signature:** `void SetDebugOutput_Studio(FILE *DebugOutput)`
- **Purpose:** Configure where loader diagnostic/status messages are written
- **Inputs:** `FILE *DebugOutput` ΓÇö stdio FILE pointer (e.g., `stderr`, `stdout`, or a log file)
- **Outputs/Return:** None
- **Side effects:** Stores a file pointer globally within the loader module
- **Calls:** None visible in header
- **Notes:** Optional configuration; allows redirecting verbose loader output to a specific stream. No validation of the pointer.

## Control Flow Notes
This is a pure declaration header with no implementation. Likely invoked during game asset-loading phase (initialization or on-demand level load). The loader parses 3DS binary format and populates Model3D's OpenGL-compatible vertex/index/bone/frame arrays.

## External Dependencies
- `<stdio.h>` ΓÇö for FILE type (debug output)
- `Model3D.h` ΓÇö defines the Model3D struct and related animation types
- `FileHandler.h` ΓÇö defines FileSpecifier abstraction for cross-platform file I/O
- Implementation module (StudioLoader.cpp or similar) ΓÇö not shown
