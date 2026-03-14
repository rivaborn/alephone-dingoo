# Source_Files/RenderOther/images.h

## File Purpose
Header for the images resource subsystem in Aleph One (Marathon engine port). Declares functions to load, manage, and render picture/sound/text resources from scenario and images files, supporting MacOS resource forks, SDL surfaces, and platform-specific optimizations.

## Core Responsibilities
- Initialize and manage images subsystem (`initialize_images_manager`)
- Check for picture existence in images file or scenario file
- Load picture/sound/text resources into `LoadedResource` wrapper objects
- Calculate and build color lookup tables (CLUTs) from picture data
- Render full-screen pictures from resource files to display
- Scroll animated picture sequences with optional text blocks
- Convert MacOS PICT resources to SDL_Surface for cross-platform rendering
- Provide surface utilities: rescale, tile, platform-specific downscaling (Dingoo)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FileSpecifier | class (external) | File path abstraction supporting MacOS FSSpec and SDL paths |
| LoadedResource | class (external) | Resource wrapper; auto-unloads on destruction |
| color_table | struct (external) | Palette/CLUT data for image rendering |
| SDL_Surface | struct (external) | SDL surface for cross-platform pixel buffers |
| CLUTSource enum | enum | Selector for color table source: Images file vs. Scenario file |

## Global / File-Static State
None. All state is managed externally via `FileSpecifier` and scenario file context.

## Key Functions / Methods

### initialize_images_manager
- Signature: `extern void initialize_images_manager(void);`
- Purpose: Initialize the images resource subsystem at engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up resource loading infrastructure
- Calls: Not inferable from this file
- Notes: Called once during engine initialization

### images_picture_exists / scenario_picture_exists
- Signature: `extern bool images_picture_exists(int base_resource);` / `extern bool scenario_picture_exists(int base_resource);`
- Purpose: Check if a picture resource with given ID exists in the respective file
- Inputs: `base_resource` (resource ID number)
- Outputs/Return: `bool` ΓÇö true if resource exists
- Side effects: File read operations
- Calls: Not inferable
- Notes: Non-destructive existence checks

### calculate_picture_clut
- Signature: `extern struct color_table *calculate_picture_clut(int CLUTSource, int pict_resource_number);`
- Purpose: Extract or compute a color lookup table from a picture resource
- Inputs: `CLUTSource` (Images/Scenario selector), `pict_resource_number` (resource ID)
- Outputs/Return: Pointer to `color_table` struct
- Side effects: Allocates palette data; caller manages lifetime
- Calls: Not inferable
- Notes: Supports two CLUT sources; returns NULL on failure (inferred)

### build_8bit_system_color_table
- Signature: `extern struct color_table *build_8bit_system_color_table(void);`
- Purpose: Create a standard 256-color palette
- Inputs: None
- Outputs/Return: Pointer to new `color_table`
- Side effects: Allocates system color table
- Calls: Not inferable
- Notes: Used as fallback when resource CLUT unavailable

### set_scenario_images_file
- Signature: `extern void set_scenario_images_file(FileSpecifier& File);`
- Purpose: Designate which scenario file contains resources to load
- Inputs: `File` (FileSpecifier reference to scenario)
- Outputs/Return: None
- Side effects: Updates internal file context for subsequent resource loads
- Calls: Not inferable
- Notes: Essential for multi-file resource management

### draw_full_screen_pict_resource_from_images / draw_full_screen_pict_resource_from_scenario
- Signature: `extern void draw_full_screen_pict_resource_from_images(int pict_resource_number);` / `extern void draw_full_screen_pict_resource_from_scenario(int pict_resource_number);`
- Purpose: Load and render a full-screen picture to the display
- Inputs: `pict_resource_number` (resource ID)
- Outputs/Return: None
- Side effects: Rendering to framebuffer
- Calls: Not inferable
- Notes: Immediate render; likely used for splash screens, menus

### scroll_full_screen_pict_resource_from_scenario
- Signature: `extern void scroll_full_screen_pict_resource_from_scenario(int pict_resource_number, bool text_block);`
- Purpose: Render an animated/scrolling picture with optional text overlay
- Inputs: `pict_resource_number` (resource ID), `text_block` (show text or not)
- Outputs/Return: None
- Side effects: Rendering to framebuffer
- Calls: Not inferable
- Notes: Used for intro sequences or story sequences

### get_picture_resource_from_images / get_picture_resource_from_scenario
- Signature: `extern bool get_picture_resource_from_images(int base_resource, LoadedResource& PictRsrc);` / `extern bool get_picture_resource_from_scenario(int base_resource, LoadedResource& PictRsrc);`
- Purpose: Load a picture resource into a `LoadedResource` wrapper (raw MacOS handle or pointer)
- Inputs: `base_resource` (resource ID), `PictRsrc` (output wrapper)
- Outputs/Return: `bool` ΓÇö success/failure
- Side effects: Loads resource into memory; wrapped object auto-unloads on destruction
- Calls: Not inferable
- Notes: Returns MacOS resource handle on Mac, pointer on SDL

### get_sound_resource_from_scenario / get_text_resource_from_scenario
- Signature: `extern bool get_sound_resource_from_scenario(int resource_number, LoadedResource& SoundRsrc);` / `extern bool get_text_resource_from_scenario(int resource_number, LoadedResource& TextRsrc);`
- Purpose: Load sound or text resource from scenario file
- Inputs: `resource_number` (ID), output `LoadedResource` reference
- Outputs/Return: `bool` ΓÇö success/failure
- Side effects: Loads resource; caller owns lifetime via wrapper
- Calls: Not inferable
- Notes: Extended to support M2-Win95 format (per commit notes)

### picture_to_surface
- Signature: `extern SDL_Surface *picture_to_surface(LoadedResource &rsrc);` (SDL only)
- Purpose: Convert MacOS PICT resource to SDL_Surface for cross-platform rendering
- Inputs: `rsrc` (LoadedResource containing PICT data)
- Outputs/Return: Pointer to new `SDL_Surface`
- Side effects: Allocates SDL surface; caller frees with SDL_FreeSurface
- Calls: Not inferable
- Notes: Conditional on SDL build

### rescale_surface / tile_surface / dingoo_downscale / dingoo_hud_downscale
- Signature: `extern SDL_Surface *rescale_surface(SDL_Surface *s, int width, int height);` etc. (SDL only)
- Purpose: Resize, tile, or platform-optimized downscale surfaces
- Inputs: Source surface, target dimensions (or none for Dingoo)
- Outputs/Return: Pointer to new `SDL_Surface`
- Side effects: Allocate new surface; caller frees
- Calls: Not inferable
- Notes: Dingoo variants are 50% downscalers optimized for that handheld platform

## Control Flow Notes
- **Startup**: `initialize_images_manager()` called during engine init
- **Resource setup**: `set_scenario_images_file()` configures scenario context
- **Rendering**: Draw/scroll functions used by UI/intro/menu subsystems
- **Resource access**: Checker functions (`*_exists`) and loader functions (`get_*_resource_from_*`) used by rendering pipelines
- **Platform support**: Conditional compilation (#ifdef SDL, #ifdef mac, #ifdef HAVE_DINGOO) branches to platform-specific implementations

## External Dependencies
- `FileHandler.h` ΓÇö provides `FileSpecifier`, `LoadedResource` classes
- Implicit: `color_table` type (defined elsewhere in codebase)
- Implicit: SDL library (conditional; `SDL_Surface`, `SDL_RWops`)
- Implicit: MacOS Carbon/Resources (on Mac builds)
- `tags.h` ΓÇö typecode constants (via FileHandler.h)

## Notes
- Header uses extern "C" semantics (C-style declarations) with C++ class references (`FileSpecifier&` pass-by-reference)
- Extensive platform branching: MacOS resource forks, generic SDL, Dingoo handheld optimization
- Resource ID system suggests centralized asset management tied to numeric constants
- Copyright notes indicate origin in Bungie Studios' Marathon (1991) with port maintenance through 2000ΓÇô2002
