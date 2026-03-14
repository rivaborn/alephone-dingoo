# Source_Files/RenderMain/OGL_Setup.cpp

## File Purpose

Central setup and initialization module for OpenGL rendering in the Aleph One game engine. Detects OpenGL availability, manages global rendering configuration, loads/unloads textures and 3D models, handles progress reporting, parses XML configuration, and provides sRGB color conversion utilities.

## Core Responsibilities

- Initialize and detect OpenGL presence on the host system
- Manage global OpenGL configuration flags, texture/model settings, fog, and color space
- Load and unload texture/model assets for game collections with progress feedback
- Parse XML configuration for fog, textures, and 3D models
- Provide color conversion wrappers with sRGB support
- Handle platform-specific OpenGL initialization (macOS AGL, SDL, Windows)
- Check for and report OpenGL extension availability

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OGL_ConfigureData` | struct | Master configuration: texture filters, model config, render flags, colors, fog, and graphics options |
| `OGL_Texture_Configure` | struct | Per-texture-type settings: near/far filters, resolution, color format, max size |
| `OGL_FogData` | struct | Fog state: RGBA color, depth, presence, landscape-affect flag |
| `OGL_TextureOptionsBase` | class | Base for texture loading: manages normal maps, glow maps, image loading from disk |
| `XML_FogParser` | class | XML parser for `<fog>` elements; handles attribute parsing and state restoration |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_OGL_IsPresent` | bool | static | Cached OpenGL detection result |
| `Using_sRGB` | bool | global | Toggle sRGB color space conversion |
| `FogData` | OGL_FogData[2] | static | Fog configs for above/below-liquid layers |
| `DefaultLscpColors` | RGBColor[4][2] | static | Landscape colors (day/night/moon/space ├ù ground/sky) |
| `OriginalFogData` | OGL_FogData* | static | Backup of original fog data for XML reset |
| `ogl_progress`, `total_ogl_progress` | int | static | Progress tracking for resource loading |
| `show_ogl_progress` | bool | static | Progress reporting enable flag |
| `last_update_tick` | int32 | static | Throttle progress updates (33ms min) |
| `FogParser`, `OpenGL_Parser` | XML_ElementParser | static | Root XML parsers for config |

## Key Functions / Methods

### OGL_Initialize
- Signature: `bool OGL_Initialize()`
- Purpose: Detect and initialize OpenGL on the host system
- Inputs: None
- Outputs/Return: bool ΓÇö true if OpenGL is available
- Side effects: Sets `_OGL_IsPresent` static variable
- Calls: `aglChoosePixelFormat()` (macOS), SDL setup (SDL target)
- Notes: Platform-specific code paths; macOS checks AGL symbol resolution; SDL assumes present

### OGL_IsPresent
- Signature: `bool OGL_IsPresent()`
- Purpose: Query cached OpenGL availability
- Inputs: None
- Outputs/Return: bool
- Side effects: None
- Calls: None (getter)

### OGL_CheckExtension
- Signature: `bool OGL_CheckExtension(const std::string extension)`
- Purpose: Check if a named OpenGL extension is supported
- Inputs: Extension name (e.g., "GL_ARB_texture_compression")
- Outputs/Return: bool ΓÇö extension present
- Side effects: None
- Calls: `glGetString(GL_EXTENSIONS)` to parse space-delimited list
- Notes: Linear scan through extension string

### OGL_SetDefaults
- Signature: `void OGL_SetDefaults(OGL_ConfigureData& Data)`
- Purpose: Populate configuration struct with sensible defaults
- Inputs: Reference to config data
- Outputs/Return: None (modifies input)
- Side effects: Fills texture filters, model config, flags, colors
- Calls: None
- Notes: Platform-specific flag defaults (SDL vs. macOS Carbon)

### OGL_TextureOptionsBase::Load
- Signature: `void OGL_TextureOptionsBase::Load()`
- Purpose: Load normal and glow texture images from disk
- Inputs: None (uses member path variables)
- Outputs/Return: None (populates `NormalImg`, `GlowImg` members)
- Side effects: File I/O, GPU memory allocation
- Calls: `glGetIntegerv()`, `OGL_CheckExtension()`, `ImageLoader` methods, `File.SetNameWithPath()`
- Notes: Enforces constraintsΓÇöglow only if normal present; same dimensions required; handles mipmaps and compression

### OGL_TextureOptionsBase::Unload
- Signature: `void OGL_TextureOptionsBase::Unload()`
- Purpose: Clear texture images from memory
- Inputs: None
- Outputs/Return: None
- Side effects: Frees `NormalImg`, `GlowImg`
- Calls: `Clear()` on image members

### OGL_TextureOptionsBase::GetMaxSize
- Signature: `int OGL_TextureOptionsBase::GetMaxSize()`
- Purpose: Retrieve max texture size for this type from config
- Inputs: None (uses `Type` member)
- Outputs/Return: int ΓÇö max size in pixels, 0 = unlimited
- Calls: `Get_OGL_ConfigureData()`

### OGL_StartProgress / OGL_ProgressCallback / OGL_StopProgress
- Purpose: Manage progress reporting during long asset loads
- `OGL_ProgressCallback` throttles updates to ~30 Hz using `SDL_GetTicks()`
- Falls back to native progress dialog if OpenGL load screen unavailable
- Calls: `OGL_LoadScreen::instance()`, progress dialog functions

### OGL_LoadModelsImages / OGL_UnloadModelsImages
- Signature: `void OGL_LoadModelsImages(short Collection)`, `void OGL_UnloadModelsImages(short Collection)`
- Purpose: Load/unload textures and 3D models for a game collection
- Inputs: Collection index (0ΓÇô31, see shape_descriptors.h)
- Outputs/Return: None
- Side effects: Allocates/frees GPU memory; modifies asset cache
- Calls: `OGL_LoadTextures()`, `OGL_UnloadTextures()`, `OGL_LoadModels()`, `OGL_UnloadModels()`
- Notes: Conditional model loading based on `OGL_Flag_3D_Models` flag

### OGL_GetFogData
- Signature: `OGL_FogData *OGL_GetFogData(int Type)`
- Purpose: Retrieve fog configuration for above- or below-liquid layer
- Inputs: Type enum (0ΓÇô1)
- Outputs/Return: Pointer to `FogData[Type]`, NULL if out of bounds
- Calls: `GetMemberWithBounds()` bounds-checking helper

### XML_FogParser :: Start / HandleAttribute / AttributesDone / ResetValues
- Purpose: Parse `<fog>` XML elements; handle "on", "depth", "landscapes", "type" attributes
- Side effects: Backs up original fog data on first parse; restores on reset
- Notes: Integrates with `Color_GetParser()` for nested color element

### OpenGL_GetParser
- Signature: `XML_ElementParser *OpenGL_GetParser()`
- Purpose: Return root XML parser for OpenGL configuration
- Inputs: None
- Outputs/Return: Pointer to `OpenGL_Parser`
- Side effects: Registers child parsers for textures, models, fog
- Calls: `AddChild()` to attach parsers

### SglColor3f, SglColor3fv, SglColor4f, SglColor4fv, etc.
- Purpose: Wrapper functions for `glColor*` with sRGB conversion
- Inputs: Color components in various formats (float, GLubyte, GLushort)
- Outputs/Return: None (calls OpenGL color functions)
- Side effects: Sets OpenGL color state after sRGB transformation
- Calls: `sRGB_frob()`, `glColor3fv()` or `glColor4fv()`
- Notes: Converts input to normalized floats; applies gamma correction if `Using_sRGB` true; alpha channels typically bypass conversion

## Control Flow Notes

**Initialization & Setup:**
1. `OGL_Initialize()` ΓåÆ detects OpenGL on startup
2. `OGL_SetDefaults()` ΓåÆ establishes baseline config
3. XML parsing (`OpenGL_GetParser()`) ΓåÆ overrides with user/mod settings

**Resource Loading:**
- `OGL_LoadModelsImages()` called per collection ΓåÆ coordinates progress via `OGL_ProgressCallback()`
- Texture and model loading delegated to external modules

**Rendering:**
- Config is read during frame rendering (not defined in this file)
- Color wrappers (`SglColor*`) apply sRGB correction at draw time if enabled

**Shutdown:**
- `OGL_UnloadModelsImages()` frees assets on level unload

## External Dependencies

- **OpenGL**: `gl.h` (GL/gl.h on SDL/Windows), `AGL.h` (macOS)
- **SDL**: `SDL_GetTicks()` for timing
- **Engine headers**:
  - `OGL_Setup.h` ΓÇö declarations
  - `OGL_LoadScreen.h` ΓÇö custom load screen rendering
  - `ColorParser.h` ΓÇö color XML parsing
  - `progress.h` ΓÇö native progress dialog
  - `shape_descriptors.h` ΓÇö collection enum definitions
  - `OGL_Win32.h` ΓÇö Windows OpenGL extensions
- **Standard library**: `<vector>`, `<string>`, `<math.h>`

**Symbols defined elsewhere:**
- `OGL_LoadTextures()`, `OGL_UnloadTextures()`, `OGL_CountTextures()`
- `OGL_LoadModels()`, `OGL_UnloadModels()`, `OGL_CountModels()`
- `OGL_IsActive()`, `OGL_ClearScreen()`
- `Get_OGL_ConfigureData()` (mutable global config reference)
- Progress dialogs: `open_progress_dialog()`, `close_progress_dialog()`, `draw_progress_bar()`
- XML helpers: `Color_GetParser()`, `TextureOptions_GetParser()`, `ModelData_GetParser()`
