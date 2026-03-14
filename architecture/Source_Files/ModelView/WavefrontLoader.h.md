# Source_Files/ModelView/WavefrontLoader.h

## File Purpose
Public interface for loading Alias Wavefront Object (.obj) format 3D models into the engine's `Model3D` structure. Provides model loading and optional debug output routing for diagnostic messages during parse operations.

## Core Responsibilities
- Declare primary model loader entry point for Wavefront files
- Provide debug message routing configuration
- Bridge file I/O (`FileSpecifier`) with 3D model storage (`Model3D`)

## Key Types / Data Structures
None.

## Global / File-Static State
None (visible here; debug output state is managed in the implementation file).

## Key Functions / Methods

### LoadModel_Wavefront
- Signature: `bool LoadModel_Wavefront(FileSpecifier& Spec, Model3D& Model);`
- Purpose: Load a Wavefront OBJ file and populate the provided Model3D structure.
- Inputs:
  - `Spec`: File specifier identifying the .obj file to load
  - `Model`: Target Model3D object to receive geometry, normals, texture coordinates
- Outputs/Return: Boolean success flag
- Side effects: Modifies the Model3D object; may parse vertex/normal/texcoord/face data from disk
- Calls: (Not inferable from header)
- Notes: Returns false on parse errors, file not found, or format issues. The Model3D is populated with vertex positions, normals, texture coordinates, and face indices as defined in the OBJ file.

### SetDebugOutput_Wavefront
- Signature: `void SetDebugOutput_Wavefront(FILE *DebugOutput);`
- Purpose: Redirect diagnostic messages (warnings, parse status) to a file stream instead of default output.
- Inputs: `DebugOutput` ΓÇô FILE pointer (e.g., stdout, stderr, or an open log file); likely NULL to disable
- Outputs/Return: None
- Side effects: Updates internal global debug output target for subsequent LoadModel_Wavefront calls
- Calls: (Not inferable from header)
- Notes: Called during initialization to configure logging; optional feature for debugging loader issues.

## Control Flow Notes
Initialization/asset-load phase. Not part of the main frame loop. Called when the engine needs to import Wavefront model files (e.g., during level loading or model import dialogs). Debug output can be enabled early in the program startup to capture loader diagnostics.

## External Dependencies
- `<stdio.h>` ΓÇô FILE type for debug output
- `Model3D.h` ΓÇô `Model3D` struct definition
- `FileHandler.h` ΓÇô `FileSpecifier` class for file I/O abstraction
