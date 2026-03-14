# Source_Files/RenderMain/OGL_Textures.h

## File Purpose
OpenGL texture manager for the Aleph One game engine (Marathon source port). Handles texture loading, format conversion, state tracking, and rendering of both normal and glow-mapped textures. Manages memory allocation, color-table conversion, and special effects like infravision and silhouette modes.

## Core Responsibilities
- Initialize, track, and destroy OpenGL texture resources across frames
- Manage texture state (allocation, usage, glow-mapping) per collection bitmap
- Convert bitmap pixel formats (16-bit ARGB ΓåÆ 32-bit RGBA) with endianness handling
- Load and cache color tables (normal and glow-mapped) from shape collections
- Handle texture coordinate scaling and offsets for sprite padding
- Implement special visual effects (infravision tinting, silhouette rendering)
- Track texture statistics (binds, setup times, memory age)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| TextureState | struct | Manages an individual texture set with ID allocation, usage tracking, glow maps, and per-frame lifecycle |
| CollBitmapTextureState | struct | Wraps texture states for all color-table indices within a collection bitmap; stores UV scale/offset for coordinate mapping |
| TextureManager | class | Main texture orchestrator; holds shape/bitmap/colortable refs, manages buffers, performs format conversion, and coordinates rendering |
| OGL_TexturesStats | struct | Statistics tracking (texture count, bind counts, setup duration, age) for performance monitoring |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| gGLTxStats | OGL_TexturesStats | global (extern) | Accumulates frame statistics on texture binds, setup times, and lifetime age |

## Key Functions / Methods

### OGL_StartTextures
- **Signature:** `void OGL_StartTextures();`
- **Purpose:** Initialize texture manager and accounting infrastructure.
- **Side effects:** Allocates texture system state; called once at engine startup.

### OGL_StopTextures
- **Signature:** `void OGL_StopTextures();`
- **Purpose:** Shut down texture accounting and free resources.
- **Side effects:** Deallocates all managed textures.

### OGL_FrameTickTextures
- **Signature:** `void OGL_FrameTickTextures();`
- **Purpose:** Per-frame housekeeping (unused texture aging, cleanup).
- **Side effects:** Updates texture usage counters; called every rendered frame.

### TextureState::Allocate
- **Signature:** `bool Allocate(short txType);`
- **Purpose:** Allocate OpenGL texture IDs for this texture set.
- **Outputs/Return:** `true` if allocation succeeded.
- **Side effects:** Generates OpenGL texture names; initializes TexGened flags.

### TextureState::Use
- **Signature:** `bool Use(int Which);`
- **Purpose:** Mark a texture (Normal or Glowing) as in-use this frame.
- **Outputs/Return:** `true` if texture is valid and marked.
- **Side effects:** Resets `unusedFrames` counter; increments IDUsage.

### TextureManager::Setup
- **Signature:** `bool Setup();`
- **Purpose:** Primary initialization; loads geometry, color tables, and optional substitute textures.
- **Inputs:** Reads public members (ShapeDesc, Texture, ShadingTables, TransferMode, etc.).
- **Outputs/Return:** `true` if setup succeeded.
- **Side effects:** Populates NormalBuffer/GlowBuffer, sets TxtrWidth/Height, configures TextureState pointer.
- **Calls:** SetupTextureGeometry, FindColorTables, PremultiplyColorTables, LoadSubstituteTexture, GetOGLTexture, PlaceTexture.

### TextureManager::RenderNormal
- **Signature:** `void RenderNormal();`
- **Purpose:** Set up and bind normal (non-glowing) texture for rendering; safe to allocate texture IDs here.
- **Side effects:** Issues OpenGL texture binding and matrix setup.

### TextureManager::RenderGlowing
- **Signature:** `void RenderGlowing();`
- **Purpose:** Set up and bind glow-map texture for rendering (additive blend).
- **Side effects:** Issues OpenGL texture binding; must be called after RenderNormal.

### ModifyCLUT
- **Signature:** `short ModifyCLUT(short TransferMode, short CLUT);`
- **Purpose:** Remap color-table index if infravision or silhouette mode is active.
- **Inputs:** TransferMode (blend type), CLUT (color-table index).
- **Outputs/Return:** Modified CLUT index.

### Convert_16to32 (inline)
- **Signature:** `GLuint Convert_16to32(uint16 InPxl);`
- **Purpose:** Convert 16-bit ARGB 1555 pixel to 32-bit RGBA 8888, respecting host endianness.
- **Inputs:** 5-bit R, 5-bit G, 5-bit B in 16-bit format.
- **Outputs/Return:** 32-bit RGBA with alpha = 0xFF.

**Infravision/Silhouette Functions:**
- `IsInfravisionActive()` ΓÇô Check if infravision effect is enabled.
- `SetInfravisionTint(short Collection, ...)` ΓÇô Configure tint color for a shape collection.
- `FindInfravisionVersion{,RGBA}(...)` ΓÇô Apply blue infravision tint to image(s).
- `FindSilhouetteVersion(...)` ΓÇô Apply silhouette effect to image.
- `LoadModelSkin(...)` ΓÇô Load a texture from an ImageDescriptor.
- `SetPixelOpacities{,RGBA}(...)` ΓÇô Configure per-pixel transparency based on options.

## Control Flow Notes
Part of the render pipeline:
1. **Init:** OGL_StartTextures() initializes manager.
2. **Per-frame housekeeping:** OGL_FrameTickTextures() ages unused textures.
3. **Texture setup:** TextureManager::Setup() loads geometry and color tables.
4. **Rendering:** RenderNormal() (binds normal textures), then RenderGlowing() (binds glow layer).
5. **Shutdown:** OGL_StopTextures() cleans up.

## External Dependencies
- **OpenGL:** GLuint, GLdouble, GLfloat, texture binding/matrix APIs
- **Custom types (defined elsewhere):** shape_descriptor, bitmap_definition, RGBColor, rgb_color, ImageDescriptor, ImageDescriptorManager, OGL_TextureOptions
- **Conditional:** Entire file gated by `#ifdef HAVE_OPENGL`
- **Color-space constants:** MAXIMUM_SHADING_TABLE_INDEXES, NUMBER_OF_OPENGL_BITMAP_SETS, ALEPHONE_LITTLE_ENDIAN (for endianness detection)

---

**Notes:**  
- Inline utilities (FiveToEight, MakeFloatColor) are helper macros for color normalization.
- TextureManager tracks both pixel buffers (NormalBuffer, GlowBuffer) and ImageDescriptorManager wrappers (NormalImage, GlowImage) for new format management.
- LowLevelShape input is a workaround to pass sprite-specific shape info without overflowing shape_descriptors.
- Landscape textures support vertical repeats (LandscapeVertRepeat field).
