# Source_Files/RenderMain/OGL_Textures.cpp

## File Purpose

OpenGL texture manager for the Aleph One game engine that handles loading, caching, and rendering textures for walls, landscapes, sprites, and 3D models. Manages texture state, memory (VRAM purging), infravision/silhouette effects, and DXTC compression support.

## Core Responsibilities

- Allocate and deallocate OpenGL texture resources via `TextureState` objects
- Manage 2D hierarchical texture state by [type][collection][bitmap][color-table]
- Track texture usage and purge unused textures after age threshold (10-20s by type)
- Load substitute/override textures and extract geometry from shape bitmaps
- Convert color tables between Marathon/platform formats and OpenGL RGBA 8888
- Apply infravision tinting and silhouette effects per collection
- Modify pixel opacity via scale/shift (Tomb Raider hack) for different texture formats
- Generate/configure mipmaps and set filter/wrapping parameters
- Support DXTC1/3/5 compressed texture formats with proper mipmap handling
- Manage landscape aspect ratio and sprite coordinate transformations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| TxtrTypeInfoData | struct | Stores OpenGL filter/format settings per texture type |
| InfravisionData | struct | Infravision tint color (RGB 0ΓÇô1) + activation flag per collection |
| TextureState | class | Tracks OpenGL texture IDs (normal/glowing), usage count, age, lifecycle |
| CollBitmapTextureState | class | Groups TextureState by color-table for one collection+bitmap |
| TextureManager | class | Main manager for loading, setup, and rendering of individual textures |
| OGL_TexturesStats | struct | Global stats: bind count, age, timing (defined in header) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| gGLTxStats | OGL_TexturesStats | global | Texture usage statistics and performance counters |
| TxtrTypeInfoList | TxtrTypeInfoData[OGL_NUMBER_OF_TEXTURE_TYPES] | static | Filter and format settings per texture type |
| ModelSkinInfo | TxtrTypeInfoData | static | Filter/format settings for 3D model skins |
| useSGISMipmaps | bool | static | GL_SGIS_generate_mipmap extension availability flag |
| IVDataList | InfravisionData[NUMBER_OF_COLLECTIONS] | static | Infravision tint config per collection (32 entries) |
| InfravisionActive | bool | static | Global infravision enable flag |
| TextureStateSets | CollBitmapTextureState*[OGL_NUMBER_OF_TEXTURE_TYPES][MAXIMUM_COLLECTIONS] | static | 2D texture state manager array |
| sgActiveTextureStates | list<TextureState*> | static | Linked list of currently-allocated TextureStates for frame ticking |

## Key Functions / Methods

### TextureState::Allocate
- **Signature:** `bool Allocate(short txType)`
- **Purpose:** Allocate OpenGL texture IDs if not already done
- **Inputs:** `txType` ΓÇö texture type constant
- **Outputs/Return:** `true` if allocation occurred; `false` if already allocated
- **Side effects:** Calls `glGenTextures(NUMBER_OF_TEXTURES, IDs)`, adds to `sgActiveTextureStates` list, increments `gGLTxStats.inUse`
- **Calls:** `glGenTextures()`
- **Notes:** Called before first use of texture; subsequent calls are no-ops

### TextureState::FrameTick
- **Signature:** `void FrameTick()`
- **Purpose:** Update texture age and purge from VRAM if unused beyond threshold
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Increments `unusedFrames`, calls `Reset()` if age threshold exceeded per type
- **Calls:** `Reset()`
- **Notes:** Walls: 300 frames (~10s); sprites: 450 frames (~15s); weapons: 600 frames (~20s); landscapes never purged. Updates `gGLTxStats.totalAge`

### TextureManager::Setup
- **Signature:** `bool Setup()`
- **Purpose:** Prepare texture for rendering; load/cache if not already present
- **Inputs:** None (uses member state: `ShapeDesc`, `TransferMode`, `TextureType`, etc.)
- **Outputs/Return:** `false` if bitmap not found; `true` on success
- **Side effects:** Loads substitute textures or extracts from shapes bitmap, finds color tables, allocates image descriptors, minifies images to fit max texture size, stores sprite transform offsets
- **Calls:** `LoadSubstituteTexture()`, `SetupTextureGeometry()`, `FindColorTables()`, `GetFakeLandscape()`, `GetOGLTexture()`, `glGetIntegerv(GL_MAX_TEXTURE_SIZE, ΓÇª)`
- **Notes:** Major texture setup pipeline; handles fake landscapes, finds/applies glow maps; stores scale/offset for later retrieval

### TextureManager::LoadSubstituteTexture
- **Signature:** `bool LoadSubstituteTexture()`
- **Purpose:** Load replacement texture from `TxtrOptsPtr->NormalImg` if available
- **Inputs:** None (reads `TxtrOptsPtr`, `TextureType`, `Landscape_AspRatExp`)
- **Outputs/Return:** `true` if substitute loaded; `false` otherwise
- **Side effects:** Sets `TxtrWidth`/`TxtrHeight`, validates power-of-2 constraints, calls `SetPixelOpacities()`
- **Calls:** `NextPowerOfTwo()`, `SetPixelOpacities()`
- **Notes:** Walls/sprites require power-of-2 in both dimensions; landscapes power-of-2 width only; landscapes set `U_Scale`/`U_Offset`

### TextureManager::RenderNormal
- **Signature:** `void RenderNormal()`
- **Purpose:** Allocate texture IDs and upload base texture to OpenGL
- **Inputs:** None (uses `TxtrStatePtr`, `NormalImage`)
- **Outputs/Return:** void
- **Side effects:** Calls `TxtrStatePtr->Allocate(TextureType)`, `TxtrStatePtr->UseNormal()`, `PlaceTexture(NormalImage)`, updates bind count stats
- **Calls:** `PlaceTexture()`
- **Notes:** Must be called first; sets up texture IDs before glow

### TextureManager::RenderGlowing
- **Signature:** `void RenderGlowing()`
- **Purpose:** Upload glow map texture if present
- **Inputs:** None (uses `TxtrStatePtr`, `GlowImage`)
- **Outputs/Return:** void
- **Side effects:** Calls `TxtrStatePtr->UseGlowing()`, `PlaceTexture(GlowImage)`, updates stats
- **Calls:** `PlaceTexture()`
- **Notes:** Called after `RenderNormal()`; skipped if no glow texture

### TextureManager::SetupTextureMatrix
- **Signature:** `void SetupTextureMatrix()`
- **Purpose:** Configure OpenGL texture matrix rotation/scale for coordinate system alignment
- **Inputs:** None (uses `TextureType`, `TxtrOptsPtr->Substitution`, `U_Scale`, `U_Offset`)
- **Outputs/Return:** void
- **Side effects:** Modifies `GL_TEXTURE` matrix; calls `glLoadIdentity()`, `glRotatef()`, `glScalef()`, `glTranslatef()`
- **Calls:** `glMatrixMode()`, `glLoadIdentity()`, `glRotatef()`, `glScalef()`, `glTranslatef()`
- **Notes:** Walls/sprites rotate 90┬░ and flip Y; landscapes scale Y by aspect ratio and translate for centering

### OGL_StartTextures
- **Signature:** `void OGL_StartTextures()`
- **Purpose:** Initialize texture system at engine startup
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Allocates and initializes `TextureStateSets[type][collection]` arrays, reads OpenGL config, checks for `GL_SGIS_generate_mipmap`
- **Calls:** `is_collection_present()`, `get_number_of_collection_bitmaps()`, `Get_OGL_ConfigureData()`
- **Notes:** Populates filter/format info from preferences for each texture type

### OGL_StopTextures
- **Signature:** `void OGL_StopTextures()`
- **Purpose:** Shut down texture system; deallocate all resources
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Deletes all `TextureStateSets` arrays, calls `OGL_Blitter::StopTextures()` and `FontSpecifier::OGL_ResetFonts()`
- **Calls:** `delete[]`

### OGL_FrameTickTextures
- **Signature:** `void OGL_FrameTickTextures()`
- **Purpose:** Update all active textures once per frame
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Iterates `sgActiveTextureStates`, calls `FrameTick()` on each
- **Calls:** `TextureState::FrameTick()` (one per active texture)
- **Notes:** Triggers VRAM purging of stale textures

### FindInfravisionVersionRGBA
- **Signature:** `void FindInfravisionVersionRGBA(short Collection, GLfloat *Color)` and `void FindInfravisionVersionRGBA(short Collection, int NumPixels, uint32 *Pixels)` (overload)
- **Purpose:** Apply infravision tint to color(s) by averaging RGB and multiplying by tint
- **Inputs:** `Collection`, `Color`/`Pixels` (data to tint), `NumPixels` (count for overload)
- **Outputs/Return:** void (modifies input in-place)
- **Side effects:** Modifies color data
- **Calls:** None
- **Notes:** Computes `(R+G+B)/3`, multiplies by `IVDataList[Collection].(Red|Green|Blue)`

### FindSilhouetteVersion
- **Signature:** `void FindSilhouetteVersion(ImageDescriptorManager &imageManager)`
- **Purpose:** Convert texture to monochrome silhouette (for invisibility effect)
- **Inputs:** `imageManager` ΓÇö texture to convert
- **Outputs/Return:** void (modifies image)
- **Side effects:** Modifies texture data in-place
- **Calls:** `FindSilhouetteVersionRGBA()`, `FindSilhouetteVersionDXTC1()`, `FindSilhouetteVersionDXTC35()`
- **Notes:** Dispatches by format (RGBA8 vs DXTC1/3/5); sets alpha to max for full opacity

### SetPixelOpacities
- **Signature:** `void SetPixelOpacities(OGL_TextureOptions& Options, ImageDescriptorManager &imageManager)`
- **Purpose:** Modify texture alpha channel via scale/shift ("Tomb Raider opacity hack")
- **Inputs:** `Options` (opacity settings: scale, shift, type), `imageManager` (texture)
- **Outputs/Return:** void (modifies image)
- **Side effects:** Modifies alpha in-place; may decompress DXTC to RGBA if needed
- **Calls:** `SetPixelOpacitiesRGBA()`, `SetPixelOpacitiesDXTC3()`, `SetPixelOpacitiesDXTC5()`, `ImageDescriptor::MakeRGBA()`
- **Notes:** Supports opacity type: default (use alpha), Avg (R+G+B)/3, Max(R,G,B); scale 0ΓÇô1, shift ΓêÆ1 to 1

### LoadModelSkin
- **Signature:** `void LoadModelSkin(ImageDescriptor& SkinImage, short Collection, short CLUT)`
- **Purpose:** Load and configure 3D model texture (skin)
- **Inputs:** `SkinImage` ΓÇö texture image, `Collection`, `CLUT` (color lookup table)
- **Outputs/Return:** void
- **Side effects:** Calls `glTexImage2D()` / `glCompressedTexImage2DARB()`, `gluBuild2DMipmaps()`, sets texture parameters
- **Calls:** `FindInfravisionVersion()`, `FindSilhouetteVersion()`, `glGetIntegerv()`, `glTexImage2D()`, `glCompressedTexImage2DARB()`, `gluBuild2DMipmaps()`, `glTexParameteri()`
- **Notes:** Handles RGBA8 and DXTC1/3/5; generates or uploads mipmaps depending on filter type and extension support

### SetInfravisionTint
- **Signature:** `bool SetInfravisionTint(short Collection, bool IsTinted, float Red, float Green, float Blue)`
- **Purpose:** Configure infravision tint for a collection
- **Inputs:** `Collection` (0ΓÇô31), `IsTinted` (enable), `Red`/`Green`/`Blue` (0ΓÇô1)
- **Outputs/Return:** `true` (always succeeds)
- **Side effects:** Updates `IVDataList[Collection]`
- **Calls:** None
- **Notes:** Applied to textures when infravision is active

### OGL_ResetTextures
- **Signature:** `void OGL_ResetTextures()`
- **Purpose:** Force unload all textures from VRAM
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `Reset()` on all TextureState objects in all collections/types, resets model skins and fonts
- **Calls:** `OGL_IsActive()`, `OGL_ResetModelSkins()`, `FontSpecifier::OGL_ResetFonts()`, `OGL_Blitter::StopTextures()`
- **Notes:** Safe when OpenGL is inactive

**Notes:** Helper color conversion functions (`MakeEightBit`, `MakeIntColor`, `MakeFloatColor`, `FindOGLColorTable`) and configuration queries (`IsLandscapeFlatColored`, `WhetherTextureFix`) summarized inline.

## Control Flow Notes

**Startup:** `OGL_StartTextures()` ΓåÆ allocates state hierarchy, reads preferences  
**Per-frame:** `OGL_FrameTickTextures()` ΓåÆ age check + VRAM purge  
**Texture use:** `TextureManager::Setup()` (load/cache) ΓåÆ `RenderNormal()` (upload) ΓåÆ `RenderGlowing()` (glow upload) ΓåÆ `SetupTextureMatrix()` (transform) ΓåÆ render ΓåÆ `RestoreTextureMatrix()` (cleanup)  
**Special effects:** `FindInfravisionVersion*()` (tint colors), `FindSilhouetteVersion*()` (monochrome), `SetPixelOpacities*()` (alpha mod)  
**Shutdown:** `OGL_StopTextures()` ΓåÆ deallocate

## External Dependencies

- **OpenGL:** `GL/gl.h`, `GL/glu.h`, `GL/glext.h`; Windows-specific `OGL_Win32.h`; macOS `AGL/agl.h`
- **SDL:** `SDL.h`, `SDL_endian.h` (endianness, color swaps)
- **Local:** `cseries.h`, `preferences.h`, `interface.h`, `render.h`, `map.h`, `collection_definition.h`, `OGL_Blitter.h`, `OGL_Setup.h`, `OGL_Render.h`, `OGL_Textures.h`
- **Defined elsewhere:** `Get_OGL_ConfigureData()`, `is_collection_present()`, `get_number_of_collection_bitmaps()`, `get_bitmap_index()`, `OGL_GetTextureOptions()`, `OGL_CheckExtension()`, `OGL_IsActive()`, `OGL_ResetModelSkins()`, `FontSpecifier::OGL_ResetFonts()`, `PlaceTexture()`, `GetFakeLandscape()`, `GetOGLTexture()`, `FindColorTables()`, `SetupTextureGeometry()`
