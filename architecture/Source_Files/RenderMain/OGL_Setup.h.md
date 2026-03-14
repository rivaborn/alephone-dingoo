# Source_Files/RenderMain/OGL_Setup.h

## File Purpose
Header for OpenGL initialization, detection, and configuration in the Aleph One game engine. Manages texture quality tiers, 3D model loading, fog rendering, color space (sRGB) handling, and XML-based configuration parsing.

## Core Responsibilities
- Detect and initialize OpenGL; query extension availability
- Manage texture configuration per type (walls, landscapes, inhabitants, weapons-in-hand)
- Load/unload models, textures, and related assets
- Configure global rendering flags (Z-buffer, fog, 3D models, HUD, fader effects, etc.)
- Handle sRGB color space conversion and gamma correction
- Track progress callbacks during resource loading
- Provide XML parsing for OpenGL configuration

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Texture_Configure` | struct | Near/far filter modes, resolution reduction, color format depth per texture type |
| `OGL_ConfigureData` | struct | Complete OpenGL rendering settings: texture configs, model config, flags, void color, landscape colors, anisotropy, multisampling, sRGB mode |
| `OGL_FogData` | struct | Fog color, depth in world units, presence flag, and whether it affects landscapes |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Using_sRGB` | bool | extern | Indicates whether sRGB color space is currently active |

## Key Functions / Methods

### OGL_Initialize
- **Signature:** `bool OGL_Initialize()`
- **Purpose:** Initializer; detects and sets up OpenGL
- **Inputs:** None
- **Outputs/Return:** bool (whether OpenGL is present)
- **Side effects:** Initializes OpenGL subsystem
- **Calls:** (not visible in this file)
- **Notes:** Engine startup entry point

### OGL_IsPresent
- **Signature:** `bool OGL_IsPresent()`
- **Purpose:** Test for OpenGL availability on host system
- **Return:** bool

### OGL_IsActive
- **Signature:** `bool OGL_IsActive()`
- **Purpose:** Test whether OpenGL is currently active for rendering
- **Return:** bool

### OGL_CheckExtension
- **Signature:** `bool OGL_CheckExtension(const std::string)`
- **Purpose:** Query whether a specific OpenGL extension is available
- **Inputs:** Extension name string (case handling not specified)
- **Return:** bool

### Get_OGL_ConfigureData
- **Signature:** `OGL_ConfigureData& Get_OGL_ConfigureData()`
- **Purpose:** Retrieve mutable reference to current OpenGL configuration
- **Return:** Reference to configuration struct

### OGL_SetDefaults
- **Signature:** `void OGL_SetDefaults(OGL_ConfigureData& Data)`
- **Purpose:** Initialize configuration struct with sensible defaults
- **Inputs:** Configuration struct reference

### OGL_CountModelsImages / OGL_LoadModelsImages / OGL_UnloadModelsImages
- **Purpose:** Query, load, and unload 3D models and associated images by collection
- **Inputs:** Collection ID (short)
- **Notes:** Declared but implemented elsewhere

### OGL_ResetTextures
- **Signature:** `void OGL_ResetTextures()`
- **Purpose:** Flush all cached textures and force reload; recovery from texture corruption
- **Notes:** Implemented in OGL_Textures.cpp

### OGL_GetFogData
- **Signature:** `OGL_FogData *OGL_GetFogData(int Type)`
- **Purpose:** Retrieve fog parameters for a given fog type (above/below liquid)
- **Inputs:** Fog type enum (OGL_Fog_AboveLiquid or OGL_Fog_BelowLiquid)
- **Return:** Pointer to fog data

### sRGB_frob (inline)
- **Signature:** `static inline float sRGB_frob(GLfloat f)`
- **Purpose:** Apply sRGB gamma correction based on EXT_framebuffer_sRGB spec
- **Inputs:** Linear float color component
- **Return:** sRGB-corrected value (or original if `Using_sRGB` is false)
- **Calls:** `std::pow()`
- **Notes:** Inverse of sRGB transfer function; not called if `Using_sRGB` is false

### SglColor3f / SglColor3fv / SglColor3ub / SglColor4f / etc. (color functions)
- **Purpose:** sRGB-aware color-setting wrappers (defined elsewhere, declared here under `#ifdef HAVE_OPENGL`)
- **Signature:** Multiple overloads matching OpenGL naming (e.g. `SglColor3f(GLfloat r, g, b)`)
- **Notes:** Gate on `HAVE_OPENGL`; replaces direct `glColor*` calls for correct sRGB handling

## Control Flow Notes
This is a configuration and initialization header. Functions are called during:
- **Startup:** `OGL_Initialize()` during engine boot
- **Configuration phase:** `Get_OGL_ConfigureData()`, `OGL_SetDefaults()` for user preferences
- **Asset loading:** `OGL_LoadModelsImages()`, `OGL_CountModelsImages()` during scene/resource load; progress tracked via `OGL_StartProgress()` / `OGL_ProgressCallback()` / `OGL_StopProgress()`
- **Rendering setup:** `OGL_GetFogData()` queried per-frame or per-scene
- **Recovery:** `OGL_ResetTextures()` called if textures corrupt

## External Dependencies
- `XML_ElementParser.h` ΓÇö XML parsing framework; `OpenGL_GetParser()` returns parser for config files
- `OGL_Subst_Texture_Def.h` ΓÇö Texture substitution & loading (cross-file include chain)
- `OGL_Model_Def.h` ΓÇö 3D model and skin data; includes `Model3D` type and model management
- `<cmath>` ΓÇö `std::pow()` for sRGB gamma in `sRGB_frob()`
- `<string>` ΓÇö STL string for extension name parameter
- Platform OpenGL headers (included transitively via `OGL_Model_Def.h`): `<OpenGL/gl.h>` (Mac), `<GL/gl.h>` (Linux), `<agl.h>` (legacy Mac)

---

**Notes:**
- Code is gated on `HAVE_OPENGL` (flexibility for platforms without OpenGL, e.g. GP2x/Dingoo)
- Platform define `OPENGL_DOESNT_COPY_ON_SWAP` (Win32, Apple+Mach) hints at swap-buffer behavior differences
- Multiple sRGB-related GL constants defined locally for compatibility with older glext.h versions
- Texture configuration allows per-type quality degradation (walls, landscapes, inhabitants, weapons)
- Some struct definitions (e.g. `OGL_ModelData`, `OGL_SkinManager`) appear both here (in `#ifdef MOVED_OUT`) and in bundled `OGL_Model_Def.h`, suggesting code refactoring/relocation in progress
