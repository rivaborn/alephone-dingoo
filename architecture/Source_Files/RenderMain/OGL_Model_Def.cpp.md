# Source_Files/RenderMain/OGL_Model_Def.cpp

## File Purpose

Manages 3D model definitions, skins, and textures for the Aleph One OpenGL renderer. Implements model loading from multiple file formats, sequence-to-model mapping, XML-based configuration parsing, and GPU-side skin/texture management.

## Core Responsibilities

- **Model lookup & caching**: Hash-table-based retrieval of model data by collection and sequence ID
- **Model loading**: Supports Wavefront OBJ, 3DS, Dim3, and QuickDraw 3D formats with multi-file support
- **3D transformations**: Calculates and applies rotation, scaling, and translation matrices during load
- **Skin/texture management**: Manages color lookup tables (CLUTs), normal/glow maps, opacity, and blend modes
- **XML configuration**: Parses `<model>`, `<skin>`, and `<seq_map>` elements to define model properties
- **GPU texture lifecycle**: Allocates/deallocates OpenGL texture IDs, tracks in-use state
- **Batch operations**: Load/unload collections of models with progress callbacks

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SequenceMapEntry` | struct | Maps Marathon engine sequence ID to model animation sequence |
| `ModelDataEntry` | struct | Stores OGL_ModelData, sequence info, and sequence mapping table |
| `ModelHashEntry` | struct | Hash table bucket entry (model index + sequence table index) |
| `OGL_SkinData` | class | Skin configuration (CLUT, opacity, normal/glow textures, blend modes) |
| `OGL_SkinManager` | struct | Container for skins; manages OpenGL texture IDs and usage tracking |
| `OGL_ModelData` | class | Model file paths, transformation parameters, Model3D object, inherits SkinManager |
| `XML_SkinDataParser` | class | XML_ElementParser for `<skin>` elements |
| `XML_SequenceMapParser` | class | XML_ElementParser for `<seq_map>` elements |
| `XML_ModelDataParser` | class | XML_ElementParser for `<model>` elements |
| `XML_MdlClearParser` | class | XML_ElementParser for `<model_clear>` directive |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DefaultModelData` | OGL_ModelData | static | Template for initializing new model data |
| `DefaultSkinData` | OGL_SkinData | static | Template for initializing new skin data |
| `MdlList[NUMBER_OF_COLLECTIONS]` | vector<ModelDataEntry>[] | static | Per-collection model data storage |
| `MdlHash[NUMBER_OF_COLLECTIONS]` | vector<ModelHashEntry>[] | static | Per-collection hash tables for fast lookup (size 256) |
| `SkinDataParser` | XML_SkinDataParser | static | Singleton XML parser for skin tags |
| `SequenceMapParser` | XML_SequenceMapParser | static | Singleton XML parser for sequence mapping tags |
| `Mdl_ClearParser` | XML_MdlClearParser | static | Singleton XML parser for model clear directives |
| `ModelDataParser` | XML_ModelDataParser | static | Singleton XML parser for model tags |

## Key Functions / Methods

### OGL_GetModelData
- **Signature**: `OGL_ModelData *OGL_GetModelData(short Collection, short Sequence, short& ModelSequence)`
- **Purpose**: Main entry point to retrieve model data; resolves sequence mapping and falls back to neutral sequence
- **Inputs**: Collection ID, Marathon physics sequence ID; returns mapped model sequence via reference
- **Outputs/Return**: Pointer to OGL_ModelData if found, NULL otherwise; sets ModelSequence
- **Side effects**: Lazy-initializes and updates hash table on misses; no I/O
- **Calls**: None directly visible
- **Notes**: Two-level lookupΓÇöchecks hash table first; on miss, does linear search through MdlList and updates hash entry. Returns NULL if model not present (checked via `ModelPresent()`).

### OGL_ModelData::Load
- **Signature**: `void OGL_ModelData::Load()`
- **Purpose**: Loads 3D model from disk, applies transformation matrices, computes bounding box
- **Inputs**: File path, model type, scale/rotation/shift parameters from member variables
- **Outputs/Return**: None; modifies `Model` member and calls `OGL_SkinManager::Load()`
- **Side effects**: I/O (file load), matrix calculation, vertex transformation in-place (for static models), GPU texture allocation via skin manager
- **Calls**: `LoadModel_Wavefront()`, `LoadModel_Studio()`, `LoadModel_Dim3()`, `LoadModel_QD3D()` (via function pointers in type dispatch); matrix helpers `MatIdentity()`, `MatMult()`, `MatScalMult()`, `MatVecMult()`; `OGL_SkinManager::Load()`
- **Notes**: Handles both static (in-place vertex transform) and animated (stores transform matrices) models. Applies 3-axis rotation in order XΓåÆYΓåÆZ, then scaling. For Dim3 format, attempts to load up to 3 files (geometry, frames, sequences) with fallback on failure.

### OGL_ModelData::Unload
- **Signature**: `void OGL_ModelData::Unload()`
- **Purpose**: Clears model geometry and unloads skins
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Deallocates Model3D geometry; calls `OGL_SkinManager::Unload()`
- **Calls**: `Model.Clear()`, `OGL_SkinManager::Unload()`
- **Notes**: Inverse of Load(); triggers GPU texture deallocation.

### OGL_SkinManager::Use
- **Signature**: `bool OGL_SkinManager::Use(short CLUT, short Which)`
- **Purpose**: Activates a skin texture, allocating a GPU texture ID if needed
- **Inputs**: CLUT (color palette ID), Which (Normal or Glowing texture index)
- **Outputs/Return**: Boolean indicating whether skin needs to be loaded
- **Side effects**: Calls `glGenTextures()` on first use; updates IDs and IDsInUse arrays; binds texture
- **Calls**: `glGenTextures()`, `glBindTexture()`
- **Notes**: Caches texture ID after first allocation. Returns true only on first call for a given CLUT/Which pair.

### OGL_SkinManager::Reset
- **Signature**: `void OGL_SkinManager::Reset(bool Clear_OGL_Txtrs)`
- **Purpose**: Resets skin texture state, optionally deallocating GPU textures
- **Inputs**: Boolean flag to clear OpenGL texture objects
- **Outputs/Return**: None
- **Side effects**: Conditionally calls `glDeleteTextures()`; clears IDsInUse array
- **Calls**: `glDeleteTextures()` (if flag set)
- **Notes**: Used when textures need to be reloaded or the context is reset.

### OGL_LoadModels / OGL_UnloadModels
- **Signature**: `void OGL_LoadModels(short Collection)`, `void OGL_UnloadModels(short Collection)`
- **Purpose**: Batch load/unload all models in a collection
- **Inputs**: Collection ID
- **Outputs/Return**: None
- **Side effects**: Calls Load()/Unload() on each ModelDataEntry; triggers progress callback
- **Calls**: `OGL_ModelData::Load()` or `OGL_ModelData::Unload()`; `OGL_ProgressCallback()`
- **Notes**: Load calls progress callback after each model; provides user feedback during startup.

### Matrix Helpers (MatCopy, MatIdentity, MatMult, MatScalMult, MatVecMult)
- **Purpose**: 3├ù3 and 3├ù4 transformation matrix operations (no OpenGL context required)
- **Notes**: Applied in sequence during Load() to compose X/Y/Z rotations. MatScalMult applied conditionally (negative scale flips normals). Results copied into Model.TransformPos and Model.TransformNorm for animated models, or applied in-place for static vertices.

### XML Parser Classes (Start, HandleAttribute, AttributesDone, End)
- **Purpose**: Parse `<model>`, `<skin>`, `<seq_map>`, and `<model_clear>` XML elements
- **Notes**: XML_ModelDataParser as root; adds children (SkinDataParser, SequenceMapParser) dynamically. End() method performs deduplicationΓÇöchecks if entry exists before insert/update. ResetValues() clears all model data on first element.

## Control Flow Notes

**Initialization**: XML configuration is parsed once at startup, populating `MdlList[]` and `MdlHash[]` with metadata. Hash table entries are lazy-initialized on first lookup.

**Runtime lookup**: `OGL_GetModelData()` is called per-entity per-frame (or per-render pass) to retrieve the appropriate model. Hash lookup is O(1) on hit; linear fallback is O(N) on miss.

**Load/Unload lifecycle**: `OGL_LoadModels()` typically called during level load (often shown with progress bar). `OGL_UnloadModels()` called during level unload. This decouples parsing (early) from GPU allocation (on-demand).

**Transformation**: Applied once during Load(); stored in Model3D for use during rendering.

## External Dependencies

- **Includes**: `cseries.h` (base types, utilities), `OGL_Model_Def.h` (class declarations), `OGL_Setup.h` (config access), `<cmath>` (trig)
- **Model loaders**: `Dim3_Loader.h`, `StudioLoader.h`, `WavefrontLoader.h`, `QD3D_Loader.h` (format-specific loaders)
- **OpenGL**: `GL/gl.h` headers included via OGL_Setup; calls `glGenTextures()`, `glBindTexture()`, `glDeleteTextures()`
- **Defined elsewhere**: `XML_ElementParser` (base class), `Model3D` class, `OGL_ProgressCallback()`, `Get_OGL_ConfigureData()`, various `ReadBoundedInt16Value()` and `ReadFloatValue()` helpers
