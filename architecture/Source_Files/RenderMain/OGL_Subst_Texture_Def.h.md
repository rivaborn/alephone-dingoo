# Source_Files/RenderMain/OGL_Subst_Texture_Def.h

## File Purpose
Defines OpenGL substitute texture options and management for wall textures and sprites in the Aleph One engine. Provides declarations for retrieving, loading/unloading textures, and XML configuration support.

## Core Responsibilities
- Define `OGL_TextureOptions` struct for substitute sprite and wall texture configuration
- Calculate and manage sprite positioning (corners relative to bitmap origin)
- Provide texture retrieval and lifecycle management (load/unload/count)
- Supply XML parsing infrastructure for texture option configuration
- Extend base texture options with sprite-specific parameters (scaling, visibility, positioning)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| OGL_TextureOptions | struct | Extends `OGL_TextureOptionsBase`; holds substitute texture settings including void visibility, sprite image scaling, and corner positioning (Left, Top, Right, Bottom) |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_TextureOptions::FindImagePosition
- **Signature:** `void FindImagePosition()`
- **Purpose:** Calculate Right and Bottom corner coordinates from Left, Top, image size, and ImageScale
- **Inputs:** (implicit) Left, Top, ImageScale, and image dimensions
- **Outputs/Return:** None (modifies Right and Bottom members)
- **Side effects:** Mutates Right and Bottom members
- **Calls:** Not inferable from this file
- **Notes:** Must be called after setting Left, Top, and ImageScale; image dimensions come from elsewhere (likely the image resource)

### OGL_TextureOptions::Original
- **Signature:** `void Original()`
- **Purpose:** Set original width and height before calculating image position
- **Inputs:** None (uses implicit state)
- **Outputs/Return:** None
- **Side effects:** Initializes original dimensions (likely for subsequent `FindImagePosition()` call)
- **Calls:** Not inferable from this file

### OGL_GetTextureOptions
- **Signature:** `OGL_TextureOptions *OGL_GetTextureOptions(short Collection, short CLUT, short Bitmap)`
- **Purpose:** Retrieve the currently active texture options for a specific collection, color table, and bitmap
- **Inputs:** Collection ID, CLUT (color lookup table) index, Bitmap index
- **Outputs/Return:** Pointer to OGL_TextureOptions (or NULL if not found)
- **Side effects:** None
- **Calls:** Not inferable from this file

### OGL_CountTextures
- **Signature:** `int OGL_CountTextures(short Collection)`
- **Purpose:** Count total textures in a collection
- **Inputs:** Collection ID
- **Outputs/Return:** Integer count of textures
- **Side effects:** None

### OGL_LoadTextures / OGL_UnloadTextures
- **Signature:** `void OGL_LoadTextures(short Collection)` / `void OGL_UnloadTextures(short Collection)`
- **Purpose:** Load or unload all textures in a collection
- **Inputs:** Collection ID
- **Outputs/Return:** None
- **Side effects:** Allocates/deallocates texture resources; likely modifies GPU state
- **Calls:** Not inferable from this file

### XML Parsing Functions
- **TextureOptions_GetParser()** and **TO_Clear_GetParser()**
  - Return `XML_ElementParser*` for parsing texture option XML elements
  - Enable MML (Marathon Markup Language) configuration of textures

## Control Flow Notes
This is a configuration/resource-definition header. Likely used during engine initialization (load textures) and runtime (retrieve options for rendering). Corresponds to resource management phases: init (load) ΓåÆ frame (use via `OGL_GetTextureOptions`) ΓåÆ shutdown (unload).

## External Dependencies
- `OGL_Texture_Def.h` ΓÇö defines `OGL_TextureOptionsBase` (parent struct), opacity/blend enums, and `ImageDescriptor`
- `XML_ElementParser.h` ΓÇö provides parser base class for XML configuration
- Conditional on `HAVE_OPENGL` preprocessor guard
