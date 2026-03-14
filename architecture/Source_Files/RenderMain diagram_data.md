# Source_Files/RenderMain/AnimatedTextures.cpp
## File Purpose
Implements animated wall textures for the Aleph One engine, reading sequences from XML configuration files. Maintains per-collection animation state (frame lists, timing, phases) and provides translation services to swap static texture descriptors with current animated frames during rendering.

## Core Responsibilities
- Manage animated texture sequences organized by collection ID
- Track and update animation state (current frame phase, tick phase within frame)
- Translate texture descriptors to current animated frame during rendering
- Parse XML configuration files to define animation sequences
- Support selective animation where only specified textures in a sequence are translated

## External Dependencies
- `<vector>` ΓÇö STL dynamic array (frame lists, animation collections)
- `cseries.h` ΓÇö Core utilities (types, macros, string functions)
- `AnimatedTextures.h` ΓÇö Public interface declarations
- `interface.h` ΓÇö Engine symbols: `shape_descriptor` type, macros (`GET_DESCRIPTOR_SHAPE`, `BUILD_DESCRIPTOR`), `get_number_of_collection_frames()`, `UNONE`, `NUMBER_OF_COLLECTIONS`
- Base class `XML_ElementParser` ΓÇö XML parsing framework (defined elsewhere)
- Helper functions `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`, `ReadUInt16Value()` ΓÇö Assumed from cseries/XML utilities

# Source_Files/RenderMain/AnimatedTextures.h
## File Purpose
Interface for the animated textures subsystem in Aleph One (Marathon engine port). Provides frame-by-frame texture animation during gameplay, allowing surfaces to cycle through animation sequences. Supports XML-based configuration.

## Core Responsibilities
- Update animation state each frame (advance animation timers/counters)
- Translate static texture descriptors to their current animated frame
- Provide XML parser for animated texture configuration

## External Dependencies
- `shape_descriptors.h`: Provides `shape_descriptor` typedef and bit-packing macros for encoding texture collection/shape/CLUT information
- `XML_ElementParser.h`: Provides XML parsing infrastructure for configuration files

# Source_Files/RenderMain/collection_definition.h
## File Purpose

Header file defining the binary data structures for game sprite and graphics collections in the Aleph One engine. Specifies layouts for color palettes, sprite animations, individual sprite frames, and associated metadata used throughout the rendering pipeline.

## Core Responsibilities

- Defines collection types enum (_wall_collection, _object_collection, _scenery_collection, etc.)
- Defines collection_definition as the root container for graphics data with offset tables and metadata
- Defines high_level_shape_definition for sprite animation state (frame counts, timing, sounds, view angles)
- Defines low_level_shape_definition for individual sprite frame rendering data (bitmap reference, coordinates, lighting, bounds)
- Defines rgb_color_value for color palette entries with flags
- Provides binary format version constants and size assertions for serialization/deserialization

## External Dependencies

- Standard library: std::vector (for dynamic arrays)
- Assumed custom types: `_fixed` (fixed-point type, defined elsewhere), transfer mode constants (likely from interface.h)
- Forward declarations: bitmap_definition, high_level_shape_definition, low_level_shape_definition (satisfy internal references)


# Source_Files/RenderMain/Crosshairs.cpp
## File Purpose
Implements crosshair rendering for the game HUD, supporting multiple shapes (traditional cross or circular octagon) with configurable size, thickness, and color. Uses Quickdraw graphics API to draw in 2D screen space, centered on the viewport.

## Core Responsibilities
- Manage global crosshairs active/inactive state via static variable
- Render crosshairs at viewport center in two shapes: RealCrosshairs (perpendicular lines) or Circle (octagon approximation)
- Preserve and restore graphics context state (pen, colors) during rendering
- Provide three rendering overloads (context + rect, context only, rect only)
- Handle pen positioning and line drawing for different crosshair geometries

## External Dependencies
- **Quickdraw API** (Mac/legacy): `GetPenState`, `SetPenState`, `RGBForeColor`, `RGBBackColor`, `PenNormal`, `PenSize`, `MoveTo`, `LineTo`, `Line`, `GetPort`, `SetPort`, `GetPortBounds`
- **cseries.h**: Core type definitions and macros
- **Crosshairs.h**: `CrosshairData` struct, shape enums (`CHShape_RealCrosshairs`, `CHShape_Circle`)
- **Preferences system** (defined elsewhere): `GetCrosshairData()` ΓÇô retrieves crosshair configuration

# Source_Files/RenderMain/Crosshairs.h
## File Purpose
Header file defining the interface for crosshair rendering in the Aleph One game engine. Provides configuration structures and functions to manage crosshair appearance, visibility state, and rendering across multiple graphics backends (classic Macintosh QuickDraw and SDL).

## Core Responsibilities
- Define `CrosshairData` structure holding visual properties (color, thickness, opacity, shape)
- Expose functions to query and toggle crosshair active state
- Provide configuration dialog for user customization
- Declare rendering entry points for multiple graphics contexts (Mac QD and SDL)
- Manage crosshair data lifecycle (retrieval from preferences)

## External Dependencies
- **Types:** `RGBColor` (color primitive, defined elsewhere), `Rect` (rectangle, likely in Mac toolbox or SDL), `GrafPtr` (Mac QuickDraw graphics port), `SDL_Surface` (SDL graphics surface)
- **Modules:** `PlayerDialogs.c` (UI dialog implementation), `preferences.c` (preference storage/retrieval)
- **Platform abstractions:** Conditional compilation (`#if defined(mac)` / `#elif defined(SDL)`) for graphics backend selection

# Source_Files/RenderMain/Crosshairs_SDL.cpp
## File Purpose
Implements SDL-based rendering of crosshairs centered on the player's viewport. Supports two crosshair shapes (traditional cross and circle approximated as octagon) with configurable color, thickness, and distance from center. Part of the Aleph One engine's heads-up display system.

## Core Responsibilities
- Manage crosshair active/inactive state via getter and setter functions
- Render crosshairs to an SDL surface at screen center
- Support RealCrosshairs shape (four-bar cross using filled rectangles)
- Support Circle shape (twelve-segment octagon approximation using line drawing)
- Map crosshair color from RGB to SDL pixel format
- Calculate crosshair dimensions based on configuration (thickness, length, distance from center)

## External Dependencies
- **SDL**: `SDL_Surface`, `SDL_Rect`, `SDL_FillRect()`, `SDL_MapRGB()`
- **Crosshairs.h**: `CrosshairData` struct, `GetCrosshairData()` function
- **screen_drawing.h**: `draw_line()` function for octagon segment drawing
- **world.h**: `world_point2d` struct for line endpoint coordinates
- **cseries.h**: Standard type definitions, SDL includes, macros

# Source_Files/RenderMain/DDS.h
## File Purpose
Defines the DirectDraw Surface (DDS) file format specification for texture loading. Provides structure and constant definitions matching the DirectX 9 DDS file format, enabling parsing and interpretation of DDS texture files in the rendering pipeline.

## Core Responsibilities
- Define DDS surface descriptor flags (DDSD_* constants) to indicate which fields are present
- Define pixel format flags (DDPF_* constants) for color/compression modes
- Define surface capability flags (DDSCAPS_*, DDSCAPS2_* constants) for texture properties
- Provide DDSURFACEDESC2 structure matching the official DDS binary format specification
- Guard against double-inclusion with __DDRAW_INCLUDED__ check

## External Dependencies
- **cstypes.h**: Provides `uint32` type definition
- Conditional guard: `__DDRAW_INCLUDED__` (skips redefinition if DirectX headers already included)

# Source_Files/RenderMain/ImageLoader.h
## File Purpose
Provides an image loader interface for the Aleph One game engine. Defines `ImageDescriptor` class for holding loaded image data (pixels, mipmaps, metadata) and utilities for loading DDS-format images with format conversion and mipmap support.

## Core Responsibilities
- Load images from DDS files with configurable flags (mipmaps, DXTC compression, resize-to-POT)
- Store pixel data and metadata (dimensions, scales, format, mipmap count)
- Convert between image formats (RGBA8 Γåö DXTC3)
- Generate and access mipmap levels
- Apply alpha premultiplication
- Manage image resizing with optional total-byte constraints
- Provide copy-on-write semantics for image descriptors via template

## External Dependencies
- `DDS.h` ΓÇö `DDSURFACEDESC2` structure (DDS file format)
- `FileHandler.h` ΓÇö `FileSpecifier`, `OpenedFile` abstractions
- `cseries.h` ΓÇö platform portability macros, `uint32` type
- `<vector>` ΓÇö (for mipmap or internal storage, not visible in header)


# Source_Files/RenderMain/ImageLoader_Macintosh.cpp
## File Purpose
macOS-specific image loader that uses QuickTime to load image files from disk and convert them to the engine's native pixel format (RGBA). Supports both color and opacity/alpha channel loading, with optional resizing to powers of two and DDS format support.

## Core Responsibilities
- Load image files via QuickTime GraphicsImporter component
- Convert QuickTime ARGB pixel data to engine RGBA format
- Handle separate color and opacity channel loading
- Support power-of-two resizing with UV scale tracking
- Validate opacity images match color image dimensions
- Premultiply alpha channel if requested
- Manage QuickTime GWorld (offscreen graphics context) allocation and cleanup

## External Dependencies
- **QuickTime (macOS):** `QuickTimeComponents.h`, `GraphicsImportComponent`, GWorld/PixMap APIs
- **macOS Frameworks:** `Carbon.h` (conditional include for Carbon.h or QuickTimeComponents.h alone)
- **Engine headers:**
  - `ImageLoader.h` ΓåÆ `ImageDescriptor` class definition, `ImageLoader_*` flags, DDS support
  - `shell.h` ΓåÆ `FileSpecifier` class, `machine_has_quicktime()`, screen_printf debugging
- **Defined elsewhere:**
  - `machine_has_quicktime()` (availability check)
  - `LoadDDSFromFile()` (DDS loader, member function)
  - `NextPowerOfTwo()` (utility)
  - `PIN()` macro (clamping)
  - `vassert()` macro (assertion with message)
  - `csprintf()` function (formatted string)

# Source_Files/RenderMain/ImageLoader_SDL.cpp
## File Purpose
SDL-based image file loading implementation for the Aleph One game engine. Loads image files from disk, converts them to 32-bit RGBA SDL surfaces, and populates ImageDescriptor objects with pixel data for use in rendering (colors) or opacity maps.

## Core Responsibilities
- Load image files via SDL_image (IMG_Load) or fallback to SDL_LoadBMP
- Handle platform-specific DDS file loading via delegation
- Resize images to power-of-two dimensions when required
- Convert loaded images to 32-bit RGBA format with endianness handling
- Extract opacity channel from grayscale images
- Manage SDL surface allocation and deallocation
- Support color and opacity image modes

## External Dependencies
- **Includes/Imports:**
  - `ImageLoader.h` ΓÇô ImageDescriptor class, image mode/flag constants
  - `FileHandler.h` ΓÇô FileSpecifier class
  - `<SDL_image.h>` (conditional `HAVE_SDL_IMAGE_H`) ΓÇô IMG_Load for JPEG, PNG, etc.
  - SDL library functions: surface creation, blitting, freeing
- **Defined elsewhere:**
  - `LoadDDSFromFile()` ΓÇô DDS-specific loader
  - `NextPowerOfTwo()` ΓÇô dimension rounding utility
  - `PIN()` ΓÇô clamping utility
  - `vassert()` ΓÇô assertion macro with formatted message
  - `temporary` ΓÇô global buffer for error messages (csprintf output)

# Source_Files/RenderMain/ImageLoader_Shared.cpp
## File Purpose
Implements image descriptor operations and DDS (DirectDraw Surface) file loading with support for DXTC texture compression formats. Provides mipmap chain management, format conversions (DXTCΓåöRGBA), DXTC decompression, and alpha premultiplication.

## Core Responsibilities
- **Mipmap management**: calculate sizes, retrieve pointers, traverse mipmap chains
- **DDS file parsing**: parse headers, validate formats, load/skip mipmap levels
- **Format handling**: detect and convert RGBA8, DXTC1, DXTC3, DXTC5 formats
- **DXTC decompression**: decompress all three DXTC variants to uncompressed RGBA
- **Buffer resizing**: reallocate pixel storage and manage memory
- **Image scaling**: downscale via mipmaps or GLU when needed
- **Alpha preprocessing**: premultiply alpha channel before upload

## External Dependencies
- **AStream.h**: `AIStreamLE` for little-endian binary parsing.
- **DDS.h**: `DDSURFACEDESC2`, `DDSD_*`, `DDPF_*`, `DDSCAPS_*` flag constants.
- **ImageLoader.h**: `ImageDescriptor` class definition.
- **SDL.h**, **SDL_endian.h**: `SDL_CreateRGBSurfaceFrom()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`, `SDL_SwapLE16/32()` for surface conversion and endian swapping.
- **OpenGL** (conditional `HAVE_OPENGL`): `gluScaleImage()`, `OGL_IsActive()` for image downscaling.
- **cstypes.h**: `uint32`, `uint16`, `uint8` type definitions; `FOUR_CHARS_TO_INT()` macro.
- **stdlib.h**, **cmath**: `std::log()` for log2 computation.

# Source_Files/RenderMain/low_level_textures.h
## File Purpose
Header file providing template-based, low-level pixel blending and texture mapping routines for software rasterization. Supports multiple pixel formats (8/16/32-bit), alpha blending modes, transparency checks, tinting, and randomization effects across horizontal and vertical polygon rendering.

## Core Responsibilities
- Pixel averaging and alpha blending for different color depths (ARGB, 565, indexed)
- Horizontal polygon line texture mapping with per-pixel shading lookup
- Landscape-optimized texture mapping (fixed-scale, row-based sampling)
- Vertical polygon line texture mapping with 4-wide batch optimization
- Tinting/color overlay application via lookup tables
- Randomization/dither effects with state-preserved PRNG
- Format-aware operations (extract/blend/compose color channels based on pixel format)
- Transparency and mask-based pixel filtering

## External Dependencies
- **SDL:** `SDL_Surface`, `SDL_PixelFormat` (accessed via `world_pixels->format`)
- **Structs (defined elsewhere):** `bitmap_definition`, `view_data`, `_horizontal_polygon_line_data`, `_vertical_polygon_data`, `_vertical_polygon_line_data`, `tint_table8`, `tint_table16`, `tint_table32`
- **Macros:** `HORIZONTAL_HEIGHT_DOWNSHIFT`, `HORIZONTAL_WIDTH_DOWNSHIFT`, `LANDSCAPE_WIDTH_BITS`, `LANDSCAPE_TEXTURE_WIDTH_DOWNSHIFT`, `FIXED_INTEGERAL_PART`, `NextLowerExponent` (function), `MAX`, `MIN`
- **Types:** `pixel8`, `pixel16`, `pixel32`, `_fixed`, `uint8`, `uint16`, `uint32`, `byte`
- **Globals:** `texture_random_seed`, `number_of_shading_tables`, `world_pixels`
- **Enums:** `_sw_alpha_off`, `_sw_alpha_fast`, `_sw_alpha_nice` (alpha blend mode selectors)

# Source_Files/RenderMain/NewRenderPlaceObjs.h
## File Purpose
File content is empty or unreadable (contains only "??"). Cannot determine purpose from available content.

## Core Responsibilities
Not inferable from this file.

## External Dependencies
None visible.

---

**Note:** The file appears to be empty or corrupted. If this file should contain code, please verify:
- File encoding and integrity
- Whether the file path is correct
- If the content was properly retrieved

# Source_Files/RenderMain/NewRenderRasterize.cpp
## File Purpose
Implements polygon rasterization and clipping for the game renderer. Transforms 3D world geometry into 2D screen coordinates, clips polygons to viewport boundaries, and manages the rendering of walls, floors, ceilings, and liquid surfaces from a preprocessed visibility tree.

## Core Responsibilities
- Traverse and render the visibility tree of polygons sorted front-to-back
- Transform world-space vertex coordinates to screen-space via perspective projection
- Clip polygons to clipping windows (viewport boundaries and portal boundaries)
- Render vertical surfaces (walls/sides) with ambient lighting and texture mapping
- Render horizontal surfaces (floors/ceilings) with texture mapping
- Render liquid surface layers with support for transparency
- Generate automap data through a simplified render pass
- Support animated textures and transfer modes (fade, invisibility, static, etc.)
- Manage rasterization windows and apply debug visualization

## External Dependencies
- **Includes (game world):** `map.h` (polygons, sides, endpoints, lines), `lightsource.h` (light intensity), `media.h` (liquid surfaces)
- **Includes (rendering):** `NewRenderRasterize.h` (class definition), `AnimatedTextures.h` (texture animation), `OGL_Setup.h` (OpenGL config, shading tables), `Rasterizer.h` (rasterizer interface)
- **Includes (base):** `cseries.h` (platform macros, types)
- **Symbols defined elsewhere:** `get_line_data()`, `get_side_data()`, `get_endpoint_data()`, `get_media_data()` (accessors), `get_shape_bitmap_and_shading_table()`, `instantiate_polygon_transfer_mode()` (texture setup), `AnimTxtr_Translate()` (animated texture), `get_light_intensity()` (lighting), `rast->texture_horizontal_polygon()`, `rast->texture_vertical_polygon()`, `rast->debug_line_v()`, `rast->debug_line_h()` (rasterizer backend), `map_polygons` (global polygon array)

# Source_Files/RenderMain/NewRenderRasterize.h
## File Purpose

Header defining the `NewRenderRasterizer` class, which coordinates rendering of visibility-culled surfaces and objects into a rasterization backend. Refactors procedural rasterization code from `render.c` into a class-based architecture with centralized view and clipping state management.

## Core Responsibilities

- Coordinate per-frame rendering of horizontal surfaces (floors/ceilings), vertical surfaces (walls), and objects/sprites
- Manage clipping windows and viewport-space transformations
- Perform geometric clipping in 2D (XY), height (Z), and 3D (XZ) coordinate systems
- Delegate actual rasterization to a pluggable `RasterizerClass` backend
- Support long-distance world coordinates via `flagged_world_point2d/3d` structures
- Generate automap/minimap rendering via `fake_render_tree()`

## External Dependencies

- **`world.h`** ΓÇö world geometry types: `world_distance`, `long_point2d`, `long_vector2d`, angle constants
- **`render.h`** ΓÇö `view_data` (camera state), render flags, viewport constants
- **`NewRenderVisTree.h`** ΓÇö `NewVisTree`, `clipping_window_data`, `portal_view_data`, `render_node_data`, `translated_endpoint_data`
- **`RenderPlaceObjs.h`** ΓÇö `RenderPlaceObjsClass`, `render_object_data`
- **`Rasterizer.h`** ΓÇö `RasterizerClass` (abstract backend)
- **`<vector>`** ΓÇö STL dynamic arrays (indirectly used via member pointers to render data)

**Symbols defined elsewhere:**
- `polygon_data`, `horizontal_surface_data`, `render_object_data` ΓÇö defined in included headers or elsewhere in render subsystem
- `bitmap_definition` ΓÇö frame/texture target (used in bundled `render.h`)

# Source_Files/RenderMain/NewRenderVisTree.cpp
## File Purpose
Implements the `NewVisTree` class, which builds a hierarchical visibility tree for a 3D game engine. The visibility tree determines which polygons are visible from the player's viewpoint, handling portal transitions and visibility culling through a queue-based algorithm that processes visible surfaces in depth order.

## Core Responsibilities
- Build visibility tree by recursively queuing visible polygons through transparent sides
- Transform map geometry into portal-local coordinate spaces using per-portal viewing parameters
- Maintain translation tables mapping world geometry indices to viewer-space indices
- Calculate screen-space clipping windows for each visible polygon
- Flatten the tree into a sorted, back-to-front list of renderable nodes
- Handle portal-within-portal viewing (nested portal views)
- Validate tree structure during debug builds

## External Dependencies
- **map.h**: `polygon_data`, `line_data`, `endpoint_data`, `side_data`, `portal_data`; reference types (`polygon_reference`, `line_reference`, etc.); global lists (`EndpointList`, `LineList`, `SideList`, `PolygonList`)
- **world.h**: `long_vector2d`, `long_point2d`, `world_point3d`, `world_distance`; angle type and trig tables (`normalize_angle`, `cosine_table`, `sine_table`)
- **render.h**: `view_data` type (not fully defined here)
- **WorkQueue.h**: `WorkQueue<T>` template
- **cseries.h**: Macros (`WRAP_HIGH`, `PIN`, `TEST_FLAG`); `NewPtr()`, `bad_alloc`
- **readout_data.h**: `gUsageTimers` performance tracking struct
- **Not defined here**: `transform_long_point2d()`, `find_line_vector_intersection()` (defined elsewhere in engine)

# Source_Files/RenderMain/NewRenderVisTree.h
## File Purpose
Defines a visibility tree class for portal-based rendering culling and clipping. Manages a hierarchical render tree of polygons, clipping windows, and portal transitions to optimize visibility determination and determine what surfaces are rendered each frame.

## Core Responsibilities
- Build and maintain hierarchical render tree from world geometry
- Translate world-space coordinates into rendering-space through portal chains
- Manage clipping windows for visibility culling at elevation lines and portals
- Track unique endpoints, lines, and polygons across portal views
- Sort render nodes back-to-front for correct draw order
- Cache computed clip data to avoid redundant calculations

## External Dependencies
- **world.h**: `angle`, `world_distance`, `world_point3d`, `long_vector2d` (coordinate and rotation types)
- **render.h**: `view_data` (camera parameters), `polygon_reference`, `endpoint_reference`, `side_reference`, `line_reference`, `portal_reference` (opaque handles to world geometry)
- **WorkQueue.h**: Template FIFO queue for `line_render_data` batching
- **Standard library**: `<vector>` (dynamic arrays)
- **Undefined here (from bundled headers)**: `rasterize_window` (base class), world/render function definitions

# Source_Files/RenderMain/OGL_Faders.cpp
## File Purpose
Implements OpenGL-based screen fading effects for the Aleph One game engine. Manages a queue of fader effects (tints, static, negation, dodge, burn) and applies them to the rendered framebuffer using various OpenGL blending modes and color operations.

## Core Responsibilities
- Manage a queue of pending fader effects and their parameters
- Implement six distinct fader types with different visual algorithms
- Handle color manipulation utilities (alpha multiplication, complement)
- Coordinate OpenGL state management for blending and color operations
- Support both sRGB-corrected and standard color rendering
- Provide configuration-driven mode selection (flat static vs. logic-op randomization)

## External Dependencies
- **fades.h**: Fader type enums (`NONE`, `_tint_fader_type`, `_randomize_fader_type`, etc.)
- **Random.h**: `GM_Random` class (KISS/LFIB4 random generators)
- **OGL_Setup.h**: `OGL_IsActive()`, `Get_OGL_ConfigureData()`, `SglColor*()` wrappers, `Using_sRGB` flag, `sRGB_frob()`, `OGL_Flag_*` constants
- **OGL_Render.h**: (included via OGL_Setup.h or indirectly)
- **OpenGL headers**: Platform-conditional (`<OpenGL/gl.h>`, `<GL/gl.h>`, `<gl.h>`)
- **cseries.h**: Base macros and types (`TEST_FLAG`, `assert`)

# Source_Files/RenderMain/OGL_Faders.h
## File Purpose
Header file providing the OpenGL renderer interface for screen fading effects (color and transparency transitions). Manages a fader queue to support different types of fades (e.g., liquid, status effects) applied during rendering.

## Core Responsibilities
- Expose fader activity check and rendering entry point
- Define fader queue categories (Liquid vs. Other visual effects)
- Provide access to the fader queue for configuration
- Render accumulated faders to a screen region with color and alpha blending

## External Dependencies
- `config.h` ΓÇö guards entire interface with `HAVE_OPENGL` (disabled on platforms without OpenGL)
- Uses standard C types: `bool`, `short`, `float`, `int`
- Likely linked against OpenGL library (not visible in header)

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

## External Dependencies

- **Includes**: `cseries.h` (base types, utilities), `OGL_Model_Def.h` (class declarations), `OGL_Setup.h` (config access), `<cmath>` (trig)
- **Model loaders**: `Dim3_Loader.h`, `StudioLoader.h`, `WavefrontLoader.h`, `QD3D_Loader.h` (format-specific loaders)
- **OpenGL**: `GL/gl.h` headers included via OGL_Setup; calls `glGenTextures()`, `glBindTexture()`, `glDeleteTextures()`
- **Defined elsewhere**: `XML_ElementParser` (base class), `Model3D` class, `OGL_ProgressCallback()`, `Get_OGL_ConfigureData()`, various `ReadBoundedInt16Value()` and `ReadFloatValue()` helpers

# Source_Files/RenderMain/OGL_Model_Def.h
## File Purpose
Defines OpenGL model data structures and management for 3D models and their skins in the Aleph One game engine. Provides configuration structures for model preprocessing, lighting, and texture handling, along with XML parsing support for model definitions.

## Core Responsibilities
- Define skin data and skin management for 3D models (color-table variants, texture IDs)
- Store model preprocessing parameters (scale, rotation, shifting, normal/lighting/depth type)
- Provide convenience load/unload methods for models and skins
- Supply global functions for querying and managing models by collection and sequence
- Support XML-based configuration parsing for model definitions

## External Dependencies
- **`OGL_Texture_Def.h`** ΓÇö `OGL_TextureOptionsBase` (base struct for skin options); `ImageDescriptor`; texture blend/opacity enums
- **`Model3D.h`** ΓÇö `Model3D` (3D geometry and animation data storage)
- **`XML_ElementParser.h`** ΓÇö `XML_ElementParser` (base class for XML parsing)
- **Platform OpenGL headers** ΓÇö `<OpenGL/gl.h>` (macOS), `<AGL/agl.h>` (Classic Mac), `<GL/gl.h>` (Linux/Windows); `GLuint`, `GLfloat`, etc.
- **`<vector>`** ΓÇö STL container for dynamic arrays

# Source_Files/RenderMain/OGL_Render.cpp
## File Purpose
Main OpenGL rendering interface for the Aleph One game engine (Marathon-based). Implements 3D scene rendering, projection matrix management, coordinate system transformations, UI rendering (crosshairs, text, HUD), and shader setup for 3D models and effects.

## Core Responsibilities
- OpenGL context lifecycle management (startup, shutdown, state initialization)
- Projection and view matrix setup and switching between 3D/2D rendering modes
- Coordinate transformation between Marathon world space, eye space, and OpenGL clip space
- 2D UI rendering: crosshairs, on-screen text, HUD elements
- 3D model rendering with multi-pass shader system (normal, glowing, static effects)
- Lighting and blending mode management
- Texture preloading to avoid runtime stalls
- Static/noise effect rendering using polygon stippling or stencil buffering
- Platform-specific rendering context handling (Mac AGL, Windows)

## External Dependencies
- **Core**: `world.h` (coordinates), `map.h` (game world), `preferences.h`, `render.h`
- **OpenGL**: Platform headers (`<GL/gl.h>`, `<OpenGL/gl.h>`, or `<AGL/agl.h>`)
- **Engine subsystems**: `OGL_Textures.h`, `OGL_Faders.h`, `ModelRenderer.h`, `Crosshairs.h`, `Logging.h`
- **Symbols defined elsewhere**: `OGL_IsPresent()`, `Get_OGL_ConfigureData()`, `load_replacement_collections()`, `OGL_StartTextures()`, `SetInfravisionTint()`, `GetCrosshairData()`, `PreloadTextures()` (declared but not defined here)

# Source_Files/RenderMain/OGL_Render.h
## File Purpose
Public interface for OpenGL rendering functionality in the Marathon/Aleph One game engine. Declares the API for screen management, view setup, object rendering, and 2D graphics integration. Separates rendering calls from configuration/parameter access (handled by OGL_Setup.h).

## Core Responsibilities
- Screen initialization, teardown, and buffer management
- View transformation and perspective configuration
- Rendering of 3D geometry (walls, sprites) and UI elements (crosshairs, text)
- Infravision tinting effects
- Integration of 2D graphics (status bar, map, terminal) with OpenGL
- Window bounds and rendering target management
- Foreground/background rendering modes (e.g., weapons in hand)

## External Dependencies
- **Includes**: `OGL_Setup.h` (configuration structures and helper functions), `config.h` (HAVE_OPENGL feature gate)
- **Undefined types** (defined elsewhere): `polygon_definition`, `rectangle_definition`, `view_data`, `Rect`, `GWorldPtr`, `CGrafPtr`, `RGBColor`
- **Preprocessing**: Gated by `HAVE_OPENGL` macro; some functions Mac-only (`#ifdef mac`)

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

# Source_Files/RenderMain/OGL_Subst_Texture_Def.cpp
## File Purpose
Implements OpenGL substitute texture management for the Aleph One game engine, including loading/unloading textures, caching texture options via hash tables, and XML parsing for texture configuration. Supports per-collection texture organization with fast lookup by CLUT and bitmap ID.

## Core Responsibilities
- Load and unload texture images from collections, computing sprite positioning
- Maintain texture options storage organized by collection with hash-table caching for O(1) lookups
- Parse XML texture configuration elements (`<texture>` and `<txtr_clear>`) into runtime data structures
- Support bulk clearing of texture entries per collection or globally
- Handle texture option replacement and insertion with CLUT-specific vs. generic (ALL_CLUTS) fallbacks

## External Dependencies
- **cseries.h** ΓÇô utility macros, types, and string parsing helpers
- **OGL_Texture_Def.h** ΓÇô `OGL_TextureOptionsBase` struct definition
- **XML_ElementParser.h** ΓÇô `XML_ElementParser` base class
- **deque, vector** ΓÇô STL containers
- **Defined elsewhere:** `OGL_ProgressCallback()`, `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `ReadBooleanValueAsBool()`, `ReadInt16Value()`, `OGL_TextureOptions::Load()`, `OGL_TextureOptions::Unload()`, `OGL_TextureOptionsBase` definition, `NUMBER_OF_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION`, XML parsing infrastructure

# Source_Files/RenderMain/OGL_Subst_Texture_Def.h
## File Purpose
Defines OpenGL substitute texture options and management for wall textures and sprites in the Aleph One engine. Provides declarations for retrieving, loading/unloading textures, and XML configuration support.

## Core Responsibilities
- Define `OGL_TextureOptions` struct for substitute sprite and wall texture configuration
- Calculate and manage sprite positioning (corners relative to bitmap origin)
- Provide texture retrieval and lifecycle management (load/unload/count)
- Supply XML parsing infrastructure for texture option configuration
- Extend base texture options with sprite-specific parameters (scaling, visibility, positioning)

## External Dependencies
- `OGL_Texture_Def.h` ΓÇö defines `OGL_TextureOptionsBase` (parent struct), opacity/blend enums, and `ImageDescriptor`
- `XML_ElementParser.h` ΓÇö provides parser base class for XML configuration
- Conditional on `HAVE_OPENGL` preprocessor guard

# Source_Files/RenderMain/OGL_Texture_Def.h
## File Purpose

Header file defining base OpenGL texture structures and enumerations for the Aleph One game engine. Provides shared configuration and metadata for wall/sprite texture substitutions and model skins in OpenGL rendering. Authored by Loren Petrich, May 2003.

## Core Responsibilities

- Define bitmap set indices (infravision, silhouette) and enumeration constants for OpenGL texture management
- Enumerate opacity types (Crisp, Flat, Avg, Max) controlling alpha channel interpretation
- Enumerate blend types (Crossfade, Add, with premultiply variants) for texture compositing
- Provide `OGL_TextureOptionsBase` struct for shared texture configuration across wall/sprite and skin textures
- Declare Load/Unload methods and size querying for texture resource lifecycle

## External Dependencies

- **Includes:** `<vector>` (STL containers), `shape_descriptors.h` (collection/shape identifiers), `ImageLoader.h` (ImageDescriptor class)
- **Conditional:** Entire file guarded by `#ifdef HAVE_OPENGL`

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

## External Dependencies

- **OpenGL:** `GL/gl.h`, `GL/glu.h`, `GL/glext.h`; Windows-specific `OGL_Win32.h`; macOS `AGL/agl.h`
- **SDL:** `SDL.h`, `SDL_endian.h` (endianness, color swaps)
- **Local:** `cseries.h`, `preferences.h`, `interface.h`, `render.h`, `map.h`, `collection_definition.h`, `OGL_Blitter.h`, `OGL_Setup.h`, `OGL_Render.h`, `OGL_Textures.h`
- **Defined elsewhere:** `Get_OGL_ConfigureData()`, `is_collection_present()`, `get_number_of_collection_bitmaps()`, `get_bitmap_index()`, `OGL_GetTextureOptions()`, `OGL_CheckExtension()`, `OGL_IsActive()`, `OGL_ResetModelSkins()`, `FontSpecifier::OGL_ResetFonts()`, `PlaceTexture()`, `GetFakeLandscape()`, `GetOGLTexture()`, `FindColorTables()`, `SetupTextureGeometry()`

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

# Source_Files/RenderMain/OGL_Win32.cpp
## File Purpose
Windows-specific OpenGL extension loader for the Aleph One game engine. Detects and initializes OpenGL ARB extensions (multitexturing, texture compression) at startup by dynamically loading function pointers via SDL, enabling optional GPU features that may not be available on all systems.

## Core Responsibilities
- Query SDL for OpenGL extension function addresses using `SDL_GL_GetProcAddress`
- Detect multitexturing support by attempting to load `glActiveTextureARB` and `glClientActiveTextureARB`
- Set the global `has_multitex` flag based on extension availability
- Load the compressed texture function pointer `glCompressedTexImage2DARB`
- Warn (via stderr) if compressed texture extension is unavailable

## External Dependencies
- **System headers:** `<windows.h>` (Windows platform types)
- **OpenGL headers:** `<GL/gl.h>`, `<GL/glext.h>` (GL types and ARB extension definitions)
- **SDL:** `<SDL.h>` for `SDL_GL_GetProcAddress` (runtime OpenGL function lookup)
- **Standard library:** `<cstdio>` implied (fprintf, stderr)
- **Symbols defined elsewhere:** `glActiveTextureARB_ptr`, `glClientActiveTextureARB_ptr`, `glCompressedTexImage2DARB_ptr`, `has_multitex` (declared in `OGL_Win32.h`, instantiated here)

# Source_Files/RenderMain/OGL_Win32.h
## File Purpose
Windows-specific OpenGL header that manages ARB extension function pointers for the rendering system. Handles runtime loading of optional OpenGL features like multitexturing and texture compression on Windows.

## Core Responsibilities
- Define function pointer types for ARB (OpenGL Architecture Review Board) extensions
- Conditionally define or declare extension function pointers based on compilation context
- Provide convenience macros that abstract function pointer access
- Flag availability of multitexturing support

## External Dependencies
- `<GL/gl.h>`, `<GL/glext.h>` ΓÇö OpenGL core and extension headers
- `setup_gl_extensions()` ΓÇö external function that populates function pointers at runtime

# Source_Files/RenderMain/Rasterizer.h
## File Purpose
Abstract base class defining the interface for rasterizer implementations in the Aleph One game engine. Provides virtual methods for view setup, foreground object rendering, and texture rasterization that subclasses (e.g., software or OpenGL renderers) override with backend-specific implementations.

## Core Responsibilities
- Define the contract for rasterizer backend implementations
- Manage view state configuration via `SetView()`
- Handle foreground rendering setup (weapons in hand, HUD objects)
- Provide texture rendering entry points for polygons and rectangles
- Support frame lifecycle with `Begin()` and `End()` methods
- Enable horizontal reflection of foreground objects

## External Dependencies
- `#include "render.h"` ΓÇô provides `view_data`, `polygon_definition`, `rectangle_definition` types and rendering constants
- `#ifdef HAVE_OPENGL` with `#include "OGL_Render.h"` ΓÇô optional OpenGL backend support (conditional compilation)
- Dependency on types defined in bundled headers (`render.h`, optionally `OGL_Render.h`)

# Source_Files/RenderMain/Rasterizer_OGL.h
## File Purpose
OpenGL-based implementation of the Rasterizer interface. This thin wrapper class delegates to C-style OpenGL rendering functions from OGL_Render.h, providing an object-oriented API for the game engine's rendering pipeline.

## Core Responsibilities
- Implement the RasterizerClass virtual interface for OpenGL rendering
- Configure view and camera parameters before frame rendering
- Route polygon and sprite rendering to underlying OGL_Render functions
- Manage foreground vs. background rendering modes (weapons, UI layers)
- Delegate actual GPU commands to C-style OGL functions

## External Dependencies
- **Rasterizer.h** ΓÇô base class RasterizerClass
- **config.h** ΓÇô `HAVE_OPENGL` preprocessor guard
- **OGL_Render.h** (implied) ΓÇô declares OGL_SetView, OGL_StartMain, OGL_EndMain, OGL_SetForeground, OGL_SetForegroundView, OGL_RenderWall, OGL_RenderSprite (defined elsewhere)
- **render.h** (via Rasterizer.h) ΓÇô likely defines polygon_definition, rectangle_definition, view_data types

# Source_Files/RenderMain/Rasterizer_SW.h
## File Purpose
Software rasterizer implementation that provides texture rendering for a game engine. Inherits from `RasterizerClass` base to plug into a polymorphic rendering pipeline. Implementations delegate to legacy `scottish_textures` module.

## Core Responsibilities
- Provide software-based rasterizer as a `RasterizerClass` subclass
- Manage view data and screen bitmap pointers for rendering
- Implement polygon texture rasterization (horizontal and vertical)
- Implement rectangle texture rasterization
- Act as an abstraction layer enabling swappable rasterizer backends (software vs. OpenGL)

## External Dependencies
- **Rasterizer.h**: Base class `RasterizerClass`
- **render.h**: (via Rasterizer.h) Likely defines view_data, polygon_definition, rectangle_definition, bitmap_definition
- **scottish_textures.c**: Contains implementation of all three texture_* methods
- External types used but not defined here: `view_data`, `bitmap_definition`, `polygon_definition`, `rectangle_definition`

# Source_Files/RenderMain/render.cpp
## File Purpose
Central coordinator of the Marathon engine's 3D rendering pipeline. Orchestrates visibility determination, polygon depth-sorting, object placement, and rasterization. Manages view camera configuration (FOV, position, orientation) and applies visual effects (explosions, teleport folds, transfer modes).

## Core Responsibilities
- Coordinate visibility-tree construction, polygon sorting, object placement, and rasterization via composition of specialized classes
- Initialize and update view data (camera position, FOV, clipping planes, pitch/yaw/roll)
- Apply render effects (fold in/out, explosions) by modifying view parameters
- Instantiate transfer modes (invisibility, fading, static, pulsating, wobbling) for sprites and surfaces
- Render the foreground weapon/HUD layer
- Allocate and initialize render subsystem memory structures
- Manage render flags for vertices, lines, and polygons (visibility, transformation, clipping)

## External Dependencies
- **map.h:** Polygon, line, endpoint, object, lightsource data structures
- **render.h:** view_data, render flags
- **lightsource.h:** Light intensity queries
- **media.h:** Liquid/media boundary checks
- **weapons.h:** Weapon display info, animation frames
- **interface.h:** Shape information, bitmap/shading table access
- **ViewControl.h:** FOV adjustment accessors
- **AnimatedTextures.h:** Animated texture support
- **OGL_Render.h:** OpenGL rasterizer interface
- **RenderVisTree.h, RenderSortPoly.h, RenderPlaceObjs.h, RenderRasterize.h:** Decomposed render subsystems
- **Rasterizer_SW.h, Rasterizer_OGL.h:** Rasterizer implementations
- **cmath, cstring, cstdlib:** Standard library (math, strings, memory)

# Source_Files/RenderMain/render.h
## File Purpose

Core rendering subsystem interface. Defines the view configuration structure (`view_data`), rendering constants/flags for visibility culling, and declares the primary render function and helper routines for effects and UI rendering.

## Core Responsibilities

- Define `view_data` structure containing all camera/viewport parameters (FOV, screen dimensions, origin, orientation, effects state)
- Manage render flags (`_polygon_is_visible_bit`, `_endpoint_is_visible_bit`, etc.) for visibility culling across map geometry
- Declare main rendering entry point (`render_view`) and initialization
- Support render effects (fold-in, fold-out, explosions)
- Provide texture transfer mode instantiation for special rendering modes
- Manage overhead map and computer interface rendering
- Control shading modes (normal, infravision)

## External Dependencies

- **world.h**: world coordinates, angles, 3D vectors/points, trig tables
- **textures.h**: `bitmap_definition` structure (frame buffer output format)
- **ViewControl.h**: FOV accessors (`View_FOV_Normal()`, `View_FOV_TunnelVision()`, etc.) and landscape options
- **scottish_textures.h**: `rectangle_definition`, `polygon_definition`, transfer mode enums, tint tables

**Symbols used but defined elsewhere**:
- `_fixed` (fixed-point type from underlying system)
- `angle` (from world.h)
- `world_point3d`, `world_vector2d` (from world.h)
- `bitmap_definition` (from textures.h)
- `uint16`, `uint32`, `short` (platform types)

# Source_Files/RenderMain/RenderPlaceObjs.cpp
## File Purpose
Implements object-placement and depth-sorting for the Marathon game renderer. Places in-game objects (sprites, monsters, items) into sorted polygons for correct depth-based rendering, computing 2D screen projections, clipping windows, and handling 3D model bounding boxes.

## Core Responsibilities
- Build and maintain a growable list of render objects in depth order
- Compute 2D screen projections from 3D world coordinates
- Sort objects by depth relative to camera and polygon visibility tree
- Calculate clipping windows for objects spanning multiple polygons
- Handle parasitic objects (objects attached to hosts, like projectiles on monsters)
- Project 3D model bounding boxes and compute lighting directions for models
- Scale object sprite information based on size flags (enlarged/tiny objects)

## External Dependencies

- **Notable includes:**
  - `cseries.h` ΓÇö utility macros (PIN, TEST_FLAG, WRAP_HIGH/WRAP_LOW, assert, etc.)
  - `map.h` ΓÇö polygon, object, endpoint, light source data structures and accessors
  - `lightsource.h` ΓÇö `get_light_intensity()`
  - `media.h` ΓÇö `get_media_data()`, media height lookups
  - `RenderPlaceObjs.h` ΓÇö class definition
  - `OGL_Setup.h` ΓÇö `OGL_GetModelData()`, 3D model structures
  - `ChaseCam.h` ΓÇö `GetChaseCamData()`
  - `player.h` ΓÇö `current_player`

- **Key external symbols (defined elsewhere):**
  - `get_object_data(index)` ΓÇö returns object from global object list
  - `get_polygon_data(index)` ΓÇö returns polygon from global map
  - `get_endpoint_data(index)` ΓÇö returns endpoint from global map
  - `get_light_intensity(index)` ΓÇö returns light intensity value
  - `extended_get_shape_information(collection, shape)` ΓÇö sprite/shape metadata
  - `OGL_GetModelData(collection, shape, &ModelSequence)` ΓÇö 3D model lookup
  - `extended_get_shape_bitmap_and_shading_table(...)` ΓÇö texture and shading lookups
  - `instantiate_rectangle_transfer_mode(...)` ΓÇö transfer mode animation setup
  - `current_player` ΓÇö global player reference
  - `cosine_table[]`, `sine_table[]` ΓÇö pre-computed trig tables (TRIG_MAGNITUDE scale)
  - `isqrt(x)` ΓÇö fast integer square root

# Source_Files/RenderMain/RenderPlaceObjs.h
## File Purpose

Defines a class for placing game inhabitants (objects/actors) in appropriate rendering order during the frame render. Works in conjunction with visibility and polygon sorting systems to determine which objects should be rendered and in what depth order. Part of the rendering pipeline after visibility determination and polygon sorting.

## Core Responsibilities

- Build and maintain a list of render objects to be drawn
- Create individual render objects with world position, lighting intensity, and opacity
- Sort render objects into a spatial tree structure for depth ordering
- Determine visible base nodes from an object's position
- Compute per-object clipping windows for occlusion against visible polygons
- Rescale shape geometry for screen projection
- Integrate with visibility tree and sorted polygon systems

## External Dependencies

- `<vector>`: STL dynamic array for `RenderObjects`
- `"world.h"`: `long_point3d`, `world_point3d`, `world_distance`, `_fixed` types
- `"interface.h"`: `shape_information_data`, rendering constants/flags
- `"render.h"`: `view_data` structure, rendering macros
- `"RenderSortPoly.h"`: `sorted_node_data`, `clipping_window_data`, `RenderSortPolyClass`, `RenderVisTreeClass` (forward)

# Source_Files/RenderMain/RenderRasterize.cpp
## File Purpose
Implements polygon clipping and rasterization for the Marathon-compatible game engine's rendering pipeline. Transforms world-space geometry into screen-space polygons, handles visibility culling via clipping windows, and dispatches final rendering to a hardware rasterizer (software or OpenGL).

## Core Responsibilities
- Iterate sorted BSP nodes and render visible geometry (walls, floors, ceilings)
- Manage horizontal (floor/ceiling) and vertical (wall) surface rendering with correct height culling
- Handle semitransparent liquid surfaces and viewer position relative to media boundaries
- Clip polygons against viewing frustum and window boundaries (xy, xz, z coordinate spaces)
- Calculate texture origins, lighting deltas, and transfer modes for each surface
- Dispatch textured polygons to a hardware rasterizer object
- Manage vertex lists and clipping state through state-machine-based polygon clipping

## External Dependencies
- **Includes:**
  - `map.h` ΓÇö polygon, line, side, endpoint, media data structures; world geometry accessors
  - `lightsource.h` ΓÇö light data and intensity queries
  - `media.h` ΓÇö media (liquid) data
  - `RenderRasterize.h` ΓÇö class definition and helper types
  - `AnimatedTextures.h` ΓÇö animated texture translation
  - `OGL_Setup.h` ΓÇö OpenGL configuration and color/bitmap helper functions
  - `preferences.h` ΓÇö graphics preferences
  - `screen.h` ΓÇö screen mode and rendering constants
- **Defined elsewhere:**
  - World accessor functions: `get_polygon_data()`, `get_media_data()`, `get_line_data()`, `get_side_data()`, `get_endpoint_data()` (from map.h)
  - `get_light_intensity()`, `get_light_data()` (from lightsource.h)
  - `AnimTxtr_Translate()` (animated texture manager)
  - `get_shape_bitmap_and_shading_table()`, `instantiate_polygon_transfer_mode()` (rendering setup)
  - `RasterizerClass` methods: `texture_horizontal_polygon()`, `texture_vertical_polygon()`, `texture_rectangle()` (dispatch to hardware)
  - Macro helpers: `TEST_RENDER_FLAG()`, `WRAP_HIGH()`, `WRAP_LOW()`, `MAX()`, `MIN()`, `PIN()`, `SGN()`, `overflow_short_to_long_2d()`

# Source_Files/RenderMain/RenderRasterize.h
## File Purpose
Defines the `RenderRasterizerClass` for converting sorted world geometry into screen-space rendering commands. Acts as the bridge between the visibility tree / polygon sorting phases and the backend rasterizer, handling clipping and coordinate transformations for horizontal surfaces (floors/ceilings), vertical surfaces (walls), and objects/sprites.

## Core Responsibilities
- Rasterize prepared geometry from the visibility tree into draw calls
- Clip horizontal polygons (floors/ceilings) in XY and Z dimensions
- Clip vertical polygons (walls) in XZ dimension
- Render floors, ceilings, walls with lighting and texture information
- Render sprites/objects with proper depth ordering and media boundary handling
- Maintain coordinate transformations between world space and clipping-aware screen space

## External Dependencies
- `<vector>` ΓÇô STL dynamic arrays
- `world.h` ΓÇô World coordinate types (`world_distance`, `long_vector2d`, point/vector structs); trigonometry; distance functions
- `render.h` ΓÇô View data (`view_data`), render flags, screen polygon/rectangle definitions
- `RenderSortPoly.h` ΓÇô `RenderSortPolyClass` for depth-sorted polygon lists
- `RenderPlaceObjs.h` ΓÇô `render_object_data` struct for sprite rendering
- `Rasterizer.h` ΓÇô Base class `RasterizerClass` (abstract backend); `SetView()`, texture/rectangle methods

# Source_Files/RenderMain/RenderSortPoly.cpp
## File Purpose
Implements polygon depth-sorting and clipping-window generation for the Marathon-like game engine's render pipeline. Decomposes a polygon visibility tree into depth-sorted order while accumulating screen-space clipping regions for each polygon.

## Core Responsibilities
- Decompose render tree via depth-first polygon traversal (removing leaves first)
- Build screen-space clipping windows for each visible polygon
- Accumulate endpoint and line clip data from parent polygon chains
- Calculate vertical clipping bounds (top/bottom) for rendering regions
- Manage sorted node data and polygon-to-sorted-node indexing

## External Dependencies
- **Includes:** `cseries.h` (utility macros, types), `map.h` (polygon/line/endpoint data, `get_polygon_data()`), `RenderSortPoly.h` (class definition)
- **External symbols:** 
  - `view_data` (contains `screen_width`)
  - `node_data`, `polygon_data`, `endpoint_clip_data`, `line_clip_data`, `clipping_window_data` (defined elsewhere in render system)
  - `RenderVisTreeClass` (visibility tree; provides `Nodes`, `EndpointClips`, `LineClips`, `ClippingWindows`, `endpoint_x_coordinates` vectors)
  - Macros: `TEST_RENDER_FLAG()`, `vassert()`, `csprintf()`, `MAX()`, `MIN()`, `POINTER_CAST()`, `NONE`, `SHRT_MAX`, `SHRT_MIN`

# Source_Files/RenderMain/RenderSortPoly.h
## File Purpose
Defines a class for sorting game world polygons into depth order during rendering. Works from visibility tree data to organize polygons with their associated render objects and clipping information for the renderer.

## Core Responsibilities
- Maintains mapping from map polygon indices to sorted render nodes
- Builds and stores clipping window data for polygon rendering
- Accumulates endpoint and line clipping information during rendering
- Resizes internal structures as polygon count changes
- Orchestrates the polygon depth-sorting process for a given view

## External Dependencies
- **Includes:**  
  `<vector>` (STL), `world.h`, `render.h`, `RenderVisTree.h`
- **Defined Elsewhere:**
  - `view_data` ΓÇö rendering view/camera state (render.h)
  - `node_data` ΓÇö visibility tree node (RenderVisTree.h)
  - `clipping_window_data`, `endpoint_clip_data`, `line_clip_data` ΓÇö clipping structures (RenderVisTree.h)
  - `render_object_data` ΓÇö render object structure (defined elsewhere, used in sorted_node_data)
  - `RenderVisTreeClass` ΓÇö visibility tree builder (RenderVisTree.h)

# Source_Files/RenderMain/RenderVisTree.cpp
## File Purpose
Implements the visibility tree builder for the Aleph One game renderer. Constructs a hierarchical tree of visible polygons from the player's viewpoint by ray-casting against map geometry, handling viewport clipping and elevation changes for correct rendering order.

## Core Responsibilities
- Build a visibility tree by breadth-first processing of polygons seen from the player's location
- Cast rays from the viewpoint to detect polygon transitions and determine visibility
- Manage screen-space clipping data (left/right/top/bottom edges and elevation changes)
- Track transformed endpoint coordinates and clipping endpoints/lines per node
- Handle long-distance calculations with overflow corrections
- Maintain a binary search tree of nodes sorted by polygon index for efficient traversal

## External Dependencies
- **map.h**: `polygon_data`, `endpoint_data`, `line_data`, `side_data`; accessors `get_polygon_data()`, etc.; macros `TEST_RENDER_FLAG()`, `SET_RENDER_FLAG()`, `ENDPOINT_IS_TRANSPARENT()`, `LINE_IS_TRANSPARENT()`, etc.
- **render.h**: `view_data`, `world_to_screen_x`, `world_to_screen_y`, perspective/transform constants
- **world.h**: `world_point2d`, `world_point3d`, `long_vector2d`, `long_point2d`, transformation functions
- **cseries.h**: Utility macros (`PIN`, `WRAP_LOW`, `WRAP_HIGH`, `SWAP`), assertions, type aliases
- STL `<vector>`: Growable arrays replacing legacy resizable lists

# Source_Files/RenderMain/RenderVisTree.h
## File Purpose
Defines the `RenderVisTreeClass` and supporting data structures for calculating rendering visibility in the Aleph One game engine. This class builds a visibility tree to determine which map polygons (sectors/walls) are visible from a viewpoint, which gates the entire rendering pipeline. It manages polygon queues, clipping data for screen boundaries, and the BSP-like traversal of the game world's polygon adjacency graph.

## Core Responsibilities
- Build and maintain a visibility tree of polygons seen from the player viewpoint
- Cast visibility rays to determine which map polygons are in the view cone
- Track and calculate clipping information for screen boundaries (left, right, top, bottom edges)
- Translate map coordinates to screen-space coordinates for visible geometry
- Manage endpoint and line clipping data to prevent off-screen rendering
- Recursively traverse the map's polygon adjacency graph during ray casting
- Maintain a polygon queue for breadth-first visibility traversal

## External Dependencies
- **Includes**: `<vector>` (STL), `world.h`, `render.h`
- **From `world.h`**: `world_point2d`, `long_vector2d` (extended precision 2D vectors to avoid overflow on long distances), world coordinate system macros (`WORLD_ONE`, `NORMALIZE_ANGLE`, etc.)
- **From `render.h`**: `view_data` (viewpoint, viewport dimensions, camera orientation, field-of-view)
- **Defined elsewhere**: All function implementations (in RenderVisTree.cpp); map/polygon definitions (from game world)

# Source_Files/RenderMain/scottish_textures.cpp
## File Purpose

Software rasterizer implementation for texture mapping polygons and rectangles in the Aleph One game engine (Marathon remake). Provides the core texture coordinate interpolation, precalculation, and multi-format rendering pipeline for walls, floors, ceilings, and landscape geometry.

## Core Responsibilities

- **Polygon texture mapping**: Render textured horizontal and vertical polygons with perspective-correct texture coordinates
- **Rectangle texture mapping**: Handle axis-aligned textured rectangles with clipping
- **Coordinate interpolation**: Build DDA tables for edge walking and texture coordinate stepping
- **Precalculation pipeline**: Pre-compute per-line shading tables and texture deltas before rendering
- **Landscape rendering**: Special case for large repeating landscape textures with configurable aspect ratios
- **Multi-format support**: 8-, 16-, and 32-bit pixel depth rendering with optional alpha blending
- **Memory management**: Allocate and manage scratch tables for coordinate and data precalculation

## External Dependencies

- `cseries.h` ΓÇô Core series headers (types, macros, assertions)
- `render.h` ΓÇô Rendering structures (`view_data`, `polygon_definition`, `rectangle_definition`, `bitmap_definition`, `point2d`); constants (`MAXIMUM_VERTICES_PER_SCREEN_POLYGON`, `MINIMUM_VERTICES_PER_SCREEN_POLYGON`)
- `Rasterizer_SW.h` ΓÇô Software rasterizer class definition
- `preferences.h` ΓÇô Graphics preferences (`graphics_preferences` global, `software_alpha_blending` mode enum)
- `SW_Texture_Extras.h` ΓÇô Software texture alpha/opacity table support
- `low_level_textures.h` ΓÇô Template implementations for actual pixel writing (`texture_horizontal_polygon_lines<>()`, `landscape_horizontal_polygon_lines<>()`, `texture_vertical_polygon_lines<>()`, `tint_vertical_polygon_lines<>()`, `randomize_vertical_polygon_lines<>()`)
- **Defined elsewhere**: `view_data`, polygon/rectangle definitions, bitmap row addresses, trigonometric lookup tables (`cosine_table`, `sine_table`), `View_GetLandscapeOptions()`, `SDL_Surface`, shading/tint table structures

# Source_Files/RenderMain/scottish_textures.h
## File Purpose
Defines core data structures and transfer modes for texture rendering in the Marathon game engine (Aleph One). Supports both polygon (wall/surface) and rectangle (sprite/HUD) rendering with multiple shading and transfer techniques. Handles color lookup tables for 8-bit, 16-bit, and 32-bit rendering modes.

## Core Responsibilities
- Define texture transfer modes (tinted, solid, textured, shadeless, static, landscape)
- Provide color/tint lookup table structures for multiple bit depths
- Define `rectangle_definition` for sprite and UI element rendering with depth and opacity
- Define `polygon_definition` for textured polygon surfaces with vertex data
- Manage shading table allocation and global rendering state
- Support OpenGL 3D model rendering integration via shape descriptors and model data pointers

## External Dependencies
- **Included:** `shape_descriptors.h` ΓÇö provides `shape_descriptor` typedef and collection/shape packing macros
- **Defined elsewhere:**
  - `bitmap_definition` ΓÇö texture bitmap data (referenced but not defined here)
  - `OGL_ModelData` ΓÇö forward-declared; holds 3D model rendering data
  - `world_point3d`, `world_vector3d`, `long_point3d`, `point2d` ΓÇö geometric types
  - `_fixed` ΓÇö fixed-point numeric type
  - `GLfloat` ΓÇö OpenGL float type
  - Pixel types: `pixel8`, `pixel16`, `pixel32`
- **Standard C types:** uint16, int16, int32, bool

# Source_Files/RenderMain/shape_definitions.h
## File Purpose
Header file defining core data structures for shape/collection management in the rendering system. Manages metadata about graphics collections (sprites, textures, shading tables) that are loaded from disk and used during rendering.

## Core Responsibilities
- Defines the `collection_header` struct that describes an in-memory graphics collection
- Tracks disk offsets and memory layouts for collections (standard and 16-bit variants)
- Maintains a global array of collection headers indexed during rendering
- Provides constants for data structure sizes and array bounds

## External Dependencies
- `collection_definition` (defined elsewhere; pointer to in-memory collection object)
- Standard C integer types: `int16`, `uint16`, `int32`, `byte`
- Macro: `MAXIMUM_COLLECTIONS` (upper bound for collection registry)

# Source_Files/RenderMain/shape_descriptors.h
## File Purpose
Defines the packed `shape_descriptor` bitfield structure and associated utility macros for referencing graphics assets in the game. Shape descriptors encode a collection ID, shape index, and color lookup table (CLUT) into a single 16-bit value to efficiently organize and retrieve sprite/shape graphics used in rendering.

## Core Responsibilities
- Define `shape_descriptor` as a 16-bit packed bitfield type
- Enumerate all 32 asset collections (weapons, monsters, scenery, landscapes, UI, etc.)
- Provide bit-field layout constants (shape: 8 bits, collection: 5 bits, CLUT: 3 bits)
- Offer extraction macros to unpack descriptor components
- Provide construction macros to assemble descriptors from collection/shape pairs

## External Dependencies
- `uint16` type (platform-defined, likely from stdint or engine headers)


# Source_Files/RenderMain/shapes.cpp
## File Purpose

Manages loading, parsing, and rendering of sprite/texture collections from a binary shapes file. Handles color management, shading tables, and infravision tinting effects. Converts stored shape data (including RLE-compressed formats) into SDL surfaces for game rendering.

## Core Responsibilities

- Load and cache shape collections from file (color tables, bitmap data, animation definitions)
- Build per-bit-depth shading tables for lighting effects and color modulation
- Parse and manage collection metadata (high/low-level shapes, bitmap offsets)
- Decompress RLE-encoded bitmap data into pixel buffers
- Convert shape data to SDL_Surface objects with proper color palettes
- Apply infravision tinting via XML-configurable per-collection color assignments
- Manage global shading tables for 16-bit and 32-bit rendering modes

## External Dependencies

- **SDL:** `SDL_RWops`, `SDL_ReadBE16/32`, `SDL_CreateRGBSurfaceFrom`, `SDL_SetColors`, `SDL_SetColorKey`, `SDL_PixelFormat`, `SDL_MapRGB`, `SDL_SwapBE16`
- **File I/O:** `OpenedFile`, `FileSpecifier` (from FileHandler.h)
- **Rendering:** `OGL_Render.h`, `OGL_LoadScreen.h` (conditional OpenGL support)
- **XML parsing:** `XML_ElementParser`, `ColorParser.h`, `Color_GetParser()`, `Color_SetArray()`
- **Data structures:** `collection_definition`, `bitmap_definition`, `rgb_color_value` (from collection_definition.h)
- **Utilities:** `byte_swapping.h`, `Packing.h`, `SW_Texture_Extras.h`, `progress.h`, `cspixels.h` (color macros)
- **Defined elsewhere:** `bit_depth` (global), `objlist_set()`, `vassert()`, `csprintf()`, shape descriptor macros (`GET_COLLECTION`, `GET_DESCRIPTOR_*`)

# Source_Files/RenderMain/SW_Texture_Extras.cpp
## File Purpose
Manages opacity tables for software-rendered textures in the Aleph One game engine. Builds per-texture lookup tables that map shading indices to opacity values, supporting per-pixel alpha blending in software rendering. Includes XML configuration parsing for texture opacity parameters.

## Core Responsibilities
- Build opacity lookup tables from bitmap shading data, supporting 16-bit and 32-bit color formats
- Manage texture collections indexed by collection and bitmap descriptors
- Support three opacity calculation modes (fully opaque, average RGB, max RGB channel)
- Parse XML texture configuration (collection, bitmap index, opacity type, scale, shift)
- Load and unload texture resources on a per-collection basis
- Provide singleton access to global texture extras manager

## External Dependencies
- **`interface.h`** ΓÇô `get_shape_bitmap_and_shading_table()`, shape/collection accessors, macros (`GET_DESCRIPTOR_COLLECTION`, `GET_DESCRIPTOR_SHAPE`, `BUILD_DESCRIPTOR`)
- **`collection_definition.h`** ΓÇô `NUMBER_OF_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION` constants
- **`render.h`** ΓÇô `number_of_shading_tables`, `MAXIMUM_SHADING_TABLE_INDEXES`
- **`scottish_textures.h`** ΓÇô Transfer mode and shading table definitions
- **SDL** ΓÇô `SDL_PixelFormat`, `SDL_GetVideoSurface()` for pixel format introspection
- **`XML_ElementParser.h`** ΓÇô XML parser base class
- **Standard library** ΓÇô `<vector>`, `std::max()`
- **`bit_depth`** (extern) ΓÇô Global color bit depth indicator

# Source_Files/RenderMain/SW_Texture_Extras.h
## File Purpose
Defines classes for managing software-rendered texture properties, including opacity tables and scaling/shifting parameters. Part of the Aleph One game engine's rendering subsystem, providing XML-loadable texture metadata organized by game asset collections.

## Core Responsibilities
- Encapsulate per-texture opacity properties (type, scale, shift, lookup table)
- Provide singleton access to texture metadata keyed by shape descriptor
- Manage texture loading/unloading per collection
- Support XML-based serialization of texture configurations
- (Conditionally compiled: disabled on Dingoo platform for binary size)

## External Dependencies
- **Includes:** `config.h`, `cseries.h`, `cstypes.h`, `shape_descriptors.h`, `XML_ElementParser.h`, `<vector>`
- **Types/Constants defined elsewhere:**
  - `shape_descriptor`, `NUMBER_OF_COLLECTIONS`, collection enum constants (from `shape_descriptors.h`)
  - `uint8`, `int` (from `cstypes.h`)
  - `XML_ElementParser` base class (from `XML_ElementParser.h`)
- **Conditional compilation:** Entire file omitted if `HAVE_DINGOO` is defined (Dingoo platform optimization)

# Source_Files/RenderMain/texturers.cpp
## File Purpose
This is a preprocessor-heavy compilation hub that generates specialized texture rendering functions by repeatedly including a template file (`low_level_textures.cpp`) with different macro configurations. It instantiates texturer code for different color depths (8, 16, 32-bit) and texture variants (normal, transparent, translucent, big textures).

## Core Responsibilities
- Provide a utility function (`NextLowerExponent`) for bit-level calculations
- Act as a template instantiation hub, configuring and including `low_level_textures.cpp` with different preprocessor defines
- Generate function variants covering:
  - 3 color depths: 8-bit, 16-bit, 32-bit
  - Transparency modes: opaque, transparent, translucent
  - Texture sizes: normal, big (larger)
  - Polygon orientations: horizontal and vertical

## External Dependencies
- `#include "texturers.h"` ΓÇö declares all function prototypes and types
- `#include "low_level_textures.cpp"` ΓÇö included repeatedly; defines the actual texture rendering functions
- Declared externals (defined elsewhere):
  - `number_of_shading_tables`, `shading_table_fractional_bits`, `shading_table_size`, `texture_random_seed` ΓÇö global state from the texture/shading subsystem

# Source_Files/RenderMain/texturers.h
## File Purpose
Declares the interface for texture rendering operations across different bit depths (8, 16, 32-bit). Defines data structures for tint/shading tables, coordinate systems for horizontal and vertical polygon scanline rendering, and function prototypes for texture fill variants (normal, transparent, translucent, tinted, randomized).

## Core Responsibilities
- Define tint table structures (color remapping) for 8/16/32-bit pixel formats
- Provide rendering parameters and macros for horizontal/vertical polygon texture rasterization
- Declare texture rendering functions supporting normal, transparent, translucent, tinted, and randomized effects
- Support dual texture sizes: standard (128├ù128) and large (256├ù256) variants
- Define function pointer types for dynamic texturer selection

## External Dependencies
- `#include "textures.h"`: Defines `bitmap_definition`, pixel types (`pixel8`, `pixel16`, `pixel32`)
- Undefined macros used: `PIXEL8_MAXIMUM_COLORS`, `PIXEL16_MAXIMUM_COMPONENT`, `PIXEL32_MAXIMUM_COMPONENT`, `TRIG_SHIFT`, `WORLD_FRACTIONAL_BITS`, `FIXED_FRACTIONAL_BITS`, `bit_depth`


# Source_Files/RenderMain/textures.cpp
## File Purpose
Core texture and bitmap pixel data management for the Aleph One game engine. Handles memory layout calculation, row-address pre-computation for both linear and RLE-compressed bitmaps, and palette-based color remapping operations.

## Core Responsibilities
- Calculate physical memory origin of bitmap pixel data given a `bitmap_definition` structure
- Pre-compute row address pointers for fast per-row pixel access (supports both linear and RLE formats)
- Apply color palette remapping tables to bitmaps (used for dynamic color/lighting effects)
- Support two bitmap encoding schemes: fixed-stride linear and variable-length RLE
- Handle column-order vs. row-order bitmap layouts
- Manage big-endian RLE format in Marathon 2 variant

## External Dependencies
- **Includes:** `cseries.h` (platform abstraction, data types), `textures.h` (bitmap_definition, prototypes)
- **External symbols used:** `pixel8`, `byte`, `int32`, `uint16` (defined in cseries/cstypes.h)
- **Conditional compilation:** `MARATHON1` and `MARATHON2` macros gate different RLE handling; `env68k` and segment pragmas are Metrowerks-specific (legacy 68k support)

# Source_Files/RenderMain/textures.h
## File Purpose
Low-level bitmap and texture data structure definitions for the Aleph One rendering system. Provides bitmap metadata layout, initialization functions, and color-remapping utilities for managing texture pixel data and rendering state.

## Core Responsibilities
- Define bitmap metadata structure (`bitmap_definition`) for texture representation
- Provide bitmap flags for marking column-order layout, transparency, and MML override state
- Calculate pixel data origin pointers from bitmap headers
- Pre-compute row address arrays for efficient scanline access
- Support color lookup table (CLUT) remapping for palette-based texture operations

## External Dependencies
- **cseries.h** ΓÇö Provides `pixel8`, `byte`, `int16`, `int32` type definitions and platform abstraction macros.
- All function implementations are defined in `textures.c` (not present in this header).

# Source_Files/RenderMain/WorkQueue.h
## File Purpose
Template class implementing a FIFO work queue with object pooling via a free list. Designed for efficient reuse of elements without repeated allocation/deallocation. Originally developed for Aleph One's vis tree generator.

## Core Responsibilities
- Manage a FIFO queue of generic template type `T` using intrusive linked-list indexing
- Maintain a free pool of pre-allocated elements for reuse via chunked allocation
- Track active queue and free list separately using indices
- Provide push/pop operations with minimal memory fragmentation
- Support pre-allocation and bulk clearing

## External Dependencies
- `#include <vector>`: Standard library dynamic array; used to store pool of `elm` nodes

---

**Notes on design:**
- Uses integer indices instead of pointers for compact storage and cache locality
- Two linked lists managed in single array: active queue and free list
- Compiler-specific optimization comment ("CW5 doesn't make this function suck donkeyballs") suggests performance-critical code tuned for CodeWarrior 5
- `Debugger()` appears to be a debug assertion/breakpoint macro (not defined in header)


