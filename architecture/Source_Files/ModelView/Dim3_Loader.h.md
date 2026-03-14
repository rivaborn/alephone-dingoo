# Source_Files/ModelView/Dim3_Loader.h

## File Purpose
Header for the Dim3 3D model format loader. Declares the primary loading function and debug configuration interface for parsing Dim3 models (supports multi-file models via multiple passes).

## Core Responsibilities
- Declare multi-pass model loading control (first pass vs. subsequent passes)
- Provide the main `LoadModel_Dim3()` entry point for loading models from files
- Configure debug output destination for loader status messages

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| LoadModelDim3_First | enum constant | Marks the initial pass; sets up the loader state |
| LoadModelDim3_Rest | enum constant | Marks subsequent passes; loads additional model data |

## Global / File-Static State
None.

## Key Functions / Methods

### LoadModel_Dim3
- Signature: `bool LoadModel_Dim3(FileSpecifier& Spec, Model3D& Model, int WhichPass)`
- Purpose: Load a Dim3 model from a file into a Model3D structure
- Inputs: 
  - `Spec`: file path abstraction
  - `Model`: target Model3D object to populate
  - `WhichPass`: `LoadModelDim3_First` or `LoadModelDim3_Rest` to control multi-pass behavior
- Outputs/Return: `bool` success/failure
- Side effects (global state, I/O, alloc): reads from file, modifies Model3D
- Notes: Supports multi-file models; first pass initializes, subsequent passes append data

### SetDebugOutput_Dim3
- Signature: `void SetDebugOutput_Dim3(FILE *DebugOutput)`
- Purpose: Configure where debug messages are printed during loading
- Inputs: FILE pointer (stdio stream)
- Outputs/Return: void
- Side effects: updates internal debug output stream (static file scope)

## Control Flow Notes
Multi-pass loading pattern: caller invokes `LoadModel_Dim3()` once with `LoadModelDim3_First`, then again with `LoadModelDim3_Rest` for each additional file. Suggests models may be split across multiple files on disk.

## External Dependencies
- `<stdio.h>` ΓÇö FILE type for debug output
- `Model3D.h` ΓÇö Model3D struct (target for loading)
- `FileHandler.h` ΓÇö FileSpecifier abstraction (file path handling)
