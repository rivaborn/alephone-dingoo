# Source_Files/RenderMain/OGL_Model_Def.h

## File Purpose
Defines OpenGL model data structures and management for 3D models and their skins in the Aleph One game engine. Provides configuration structures for model preprocessing, lighting, and texture handling, along with XML parsing support for model definitions.

## Core Responsibilities
- Define skin data and skin management for 3D models (color-table variants, texture IDs)
- Store model preprocessing parameters (scale, rotation, shifting, normal/lighting/depth type)
- Provide convenience load/unload methods for models and skins
- Supply global functions for querying and managing models by collection and sequence
- Support XML-based configuration parsing for model definitions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OGL_SkinData` | struct | Extends texture options with CLUT selector; stores opacity/blending/image data per skin variant |
| `OGL_SkinManager` | struct | Manages multiple skins for a single model; stores texture IDs per CLUT and bitmap set; handles skin selection and loading |
| `OGL_ModelData` | class | Extends `OGL_SkinManager`; wraps a `Model3D` with preprocessing transforms (scale, rotation, shift), normal/lighting/depth configuration |
| Model light mode enum | enum (anonymous) | Defines lighting calculation types: fast, per-vertex, with/without fade |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_SkinData::GetMaxSize
- Signature: `int GetMaxSize()`
- Purpose: Determine maximum texture size for this skin variant
- Inputs: None (member function)
- Outputs/Return: Maximum size as integer
- Side effects: None
- Calls: Not visible in this file
- Notes: Virtual; likely overrides base class method

### OGL_SkinManager::Reset
- Signature: `void Reset(bool Clear_OGL_Txtrs)`
- Purpose: Reset skins for reloading; optionally clear associated OpenGL texture IDs
- Inputs: `Clear_OGL_Txtrs` ΓÇö whether to deallocate OpenGL texture objects
- Outputs/Return: None
- Side effects: Clears or resets texture IDs, invalidates `IDsInUse` flags
- Calls: Not visible in this file
- Notes: Called before reloading model skins

### OGL_SkinManager::GetSkin
- Signature: `OGL_SkinData *GetSkin(short CLUT)`
- Purpose: Retrieve skin data for a specific color-table variant
- Inputs: `CLUT` ΓÇö color lookup table index (-1 for universal skin)
- Outputs/Return: Pointer to `OGL_SkinData` or NULL if unavailable
- Side effects: None
- Calls: Not visible in this file
- Notes: Returns pointer into internal `SkinData` vector

### OGL_SkinManager::Use
- Signature: `bool Use(short CLUT, short Which)`
- Purpose: Activate a skin and determine if it needs loading
- Inputs: `CLUT` ΓÇö color table variant; `Which` ΓÇö texture type (Normal or Glowing)
- Outputs/Return: true if texture should be loaded, false otherwise
- Side effects: Updates `IDsInUse` flags; may switch active OpenGL texture
- Calls: Not visible in this file
- Notes: Guides texture loading workflow

### OGL_ModelData::ModelPresent
- Signature: `bool ModelPresent()`
- Purpose: Inline check whether a 3D model is loaded
- Inputs: None (member function)
- Outputs/Return: true if `Model.VertIndices` is non-empty
- Side effects: None
- Calls: None
- Notes: Quick validity check before rendering

### Global functions (declarations only)

**OGL_GetModelData**
- Signature: `OGL_ModelData *OGL_GetModelData(short Collection, short Sequence, short& ModelSequence)`
- Purpose: Retrieve model data for a game collection and sequence; determine actual model sequence index used
- Inputs: `Collection`, `Sequence`; output ref param `ModelSequence`
- Outputs/Return: Pointer to model data or NULL if not found
- Side effects: Populates `ModelSequence` with resolved model sequence
- Calls: Not visible in this file

**OGL_ResetModelSkins**
- Signature: `void OGL_ResetModelSkins(bool Clear_OGL_Txtrs)`
- Purpose: Reset skins for all loaded models
- Inputs: `Clear_OGL_Txtrs`
- Outputs/Return: None
- Side effects: Calls `Reset()` on all managed skins

**OGL_CountModels, OGL_LoadModels, OGL_UnloadModels**
- Signatures: `int OGL_CountModels(short Collection)`; `void OGL_LoadModels(short Collection)`; `void OGL_UnloadModels(short Collection)`
- Purpose: Query model count and manage load state for a collection
- Inputs: `Collection` index
- Outputs/Return: Count (first function only)
- Side effects: Load/unload models; manage texture and model resources
- Calls: Not visible in this file

**ModelData_GetParser, Mdl_Clear_GetParser**
- Signature: `XML_ElementParser *ModelData_GetParser()`; `XML_ElementParser *Mdl_Clear_GetParser()`
- Purpose: Return XML parsers for model definition and clear-model XML elements
- Inputs: None
- Outputs/Return: Parser object pointers
- Side effects: None (allocations handled elsewhere)
- Calls: Not visible in this file

## Control Flow Notes
This is a pure definition header; no control flow is implemented. Usage pattern:
1. Game engine queries models by collection/sequence via `OGL_GetModelData()`
2. Models are loaded/unloaded per collection via `OGL_LoadModels()` / `OGL_UnloadModels()`
3. Skins are selected at render time via `OGL_SkinManager::Use()` and applied to active OpenGL texture units
4. XML configuration is parsed at init time via the returned parser objects

## External Dependencies
- **`OGL_Texture_Def.h`** ΓÇö `OGL_TextureOptionsBase` (base struct for skin options); `ImageDescriptor`; texture blend/opacity enums
- **`Model3D.h`** ΓÇö `Model3D` (3D geometry and animation data storage)
- **`XML_ElementParser.h`** ΓÇö `XML_ElementParser` (base class for XML parsing)
- **Platform OpenGL headers** ΓÇö `<OpenGL/gl.h>` (macOS), `<AGL/agl.h>` (Classic Mac), `<GL/gl.h>` (Linux/Windows); `GLuint`, `GLfloat`, etc.
- **`<vector>`** ΓÇö STL container for dynamic arrays
