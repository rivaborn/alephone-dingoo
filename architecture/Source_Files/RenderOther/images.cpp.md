# Source_Files/RenderOther/images.cpp

## File Purpose
Manages loading, decompressing, and displaying image resources (pictures and color tables) from the game's resource and WAD files. Handles picture format conversion between Mac PICT format and SDL surfaces, with support for RLE decompression and various color depths.

## Core Responsibilities
- Load and manage PICT/CLUT resources from both MacOS resource forks and Win32-compatible WAD files
- Decompress PackBits RLE-encoded picture data at 8, 16, and 32-bit depths
- Convert MacOS PICT resources to SDL surfaces for rendering
- Handle color table (CLUT) resource management and color model conversion
- Provide high-level API for retrieving and displaying full-screen pictures with optional scrolling
- Support resource ID mapping based on display bit depth (8/16/32-bit fallback logic)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| image_file_t | class | Manages an open image file (resource fork or WAD), proxies resource retrieval |
| pict_head | struct (Mac) | MacOS PICT header: bounds rect, pixel depth, followed by pixel data |
| clut_record | struct (Mac) | MacOS color lookup table: count, flags, ID, and 256 RGB entries |
| wad_data | struct | WAD archive data structure; contains tag array for different resource types |
| LoadedResource | class (external) | Wrapper for loaded resource data with auto-cleanup |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| ImagesFile | image_file_t | static | The main Images resource file; initialized at startup, closed at shutdown |
| ScenarioFile | image_file_t | static | Per-map scenario/resource file; opened/closed per scenario |
| interface_bit_depth | short | external | Current display bit depth (8, 16, or 32); drives resource ID selection |
| draw_clip_rect_active | bool | external | Whether a clipping rectangle is active (from screen_drawing_sdl.cpp) |
| draw_clip_rect | screen_rectangle | external | Current clipping rectangle bounds |

## Key Functions / Methods

### initialize_images_manager
- **Signature:** `void initialize_images_manager(void)`
- **Purpose:** Initialize the image manager and open the main Images resource file at engine startup.
- **Inputs:** None
- **Outputs/Return:** None (fatal error if Images file not found)
- **Side effects:** Opens ImagesFile; registers atexit handler for shutdown
- **Calls:** `ImagesFile.open_file()`, `alert_user()`, `atexit()`

### image_file_t::open_file
- **Signature:** `bool open_file(FileSpecifier &file)`
- **Purpose:** Open an image file as either a MacOS resource fork or a Win32 WAD file.
- **Inputs:** FileSpecifier pointing to the file to open
- **Outputs/Return:** `true` if opened successfully (either format); `false` otherwise
- **Side effects:** Opens rsrc_file (resource fork) or wad_file + reads wad_hdr; can succeed with just one
- **Calls:** `file.Open()`, `open_wad_file_for_reading()`, `read_wad_header()`
- **Notes:** Attempts resource fork first; if that fails, tries WAD format. Both can be open simultaneously.

### image_file_t::get_rsrc
- **Signature:** `bool get_rsrc(uint32 rsrc_type, uint32 wad_type, int id, LoadedResource &rsrc)`
- **Purpose:** Retrieve a resource from either the resource file or WAD file with format conversion.
- **Inputs:** Resource type (Mac 4-char code), WAD type, resource ID, output LoadedResource
- **Outputs/Return:** `true` if resource loaded; `false` if not found
- **Side effects:** Allocates and copies resource data; may call make_rsrc_from_pict/clut for WADΓåÆPICT conversion
- **Calls:** `rsrc_file.Get()`, `read_indexed_wad_from_file()`, `extract_type_from_wad()`, `make_rsrc_from_pict()`, `make_rsrc_from_clut()`, `free_wad()`
- **Notes:** Handles WADΓåÆPICT conversion for compatibility; supports PICT, CLUT, sound, and text.

### picture_to_surface
- **Signature:** `SDL_Surface *picture_to_surface(LoadedResource &rsrc)`
- **Purpose:** Parse a MacOS PICT resource and convert it to an SDL surface.
- **Inputs:** LoadedResource containing PICT data
- **Outputs/Return:** Newly allocated SDL_Surface (RGB 8/16/32-bit); NULL on error
- **Side effects:** Allocates SDL surface and pixel memory; interprets PICT opcodes
- **Calls:** `SDL_RWFromMem()`, `SDL_CreateRGBSurface()`, `uncompress_picture()`, `SDL_SetColors()`, `SDL_BlitSurface()`, `IMG_LoadTyped_RW()` (JPEG support)
- **Notes:** Parses only the first image in PICT stream (skips subsequent CopyBits opcodes). Supports JPEG compressed images via SDL_image if HAVE_SDL_IMAGE defined.

### uncompress_picture
- **Signature:** `static int uncompress_picture(const uint8 *src, int row_bytes, uint8 *dst, int dst_pitch, int depth, int height, int pack_type)`
- **Purpose:** Decompress PackBits RLE-encoded picture data, with optional color expansion for sub-byte depths.
- **Inputs:** Compressed data pointer, bytes per row, destination buffer, pitch, pixel depth, height, pack type
- **Outputs/Return:** Number of bytes consumed from source; -1 on error
- **Side effects:** Writes decompressed pixel data to dst; allocates temporary buffer for sub-8-bit color expansion
- **Calls:** `uncompress_rle8()`, `uncompress_rle16()`, `uncompress_rle32()`, `byte_swap_memory()`, `malloc()`, `free()`
- **Notes:** Handles pack_type 0/1/3/4; expands 1/2/4-bit data to 8-bit; performs byte-swapping for big-endian formats.

### unpack_bits (template)
- **Signature:** `template <class T> static const uint8 *unpack_bits(const uint8 *src, int row_bytes, T *dst)`
- **Purpose:** Unpack a single scanline of PackBits RLE data (generic for 1/2/4-byte pixels).
- **Inputs:** Source RLE data, row byte count, destination pixel buffer
- **Outputs/Return:** Updated source pointer (past consumed data)
- **Side effects:** Writes unpacked pixels to dst; advances both src and dst
- **Calls:** None
- **Notes:** Handles both run-length and literal runs; byte-swaps 16/32-bit values on read.

### draw_picture
- **Signature:** `static void draw_picture(LoadedResource &PictRsrc)`
- **Purpose:** Display a PICT resource full-screen with optional scrolling if larger than screen.
- **Inputs:** LoadedResource containing PICT data
- **Outputs/Return:** None
- **Side effects:** Converts PICT to surface, blits to video surface, polls events, frees surface
- **Calls:** `picture_to_surface()`, `rescale_surface()`, `SDL_BlitSurface()`, `SDL_UpdateRects()`, `SDL_Delay()`, `global_idle_proc()`, `SDL_FreeSurface()`
- **Notes:** Scrolls if picture exceeds screen dimensions; aborts on click/key; horizontal/vertical scroll controlled by picture aspect ratio.

### determine_pict_resource_id
- **Signature:** `int determine_pict_resource_id(int base_id, int delta16, int delta32)`
- **Purpose:** Select the appropriate resource ID based on display bit depth, with fallback to lower depths.
- **Inputs:** Base ID, delta for 16-bit, delta for 32-bit
- **Outputs/Return:** Actual resource ID to load (base_id + delta16/delta32 or base_id for 8-bit)
- **Side effects:** Queries rsrc_file for existence of IDs
- **Calls:** `has_pict()`
- **Notes:** Tries 32ΓåÆ16ΓåÆ8 bit depth order; returns 8-bit version if nothing found.

### make_rsrc_from_pict (SDL version)
- **Signature:** `bool make_rsrc_from_pict(void *data, size_t length, LoadedResource &rsrc, void *clut_data, size_t clut_length)`
- **Purpose:** Convert a Win32 WAD-format picture tag to a MacOS PICT resource structure (SDL port).
- **Inputs:** WAD picture data, length, optional WAD CLUT data, output LoadedResource
- **Outputs/Return:** `true` if conversion succeeded; `false` if format invalid
- **Side effects:** Allocates PICT output buffer; wraps WAD picture in PICT header and opcodes
- **Calls:** `malloc()`, `memset()`, `memcpy()`
- **Notes:** Constructs a minimal PICT with HeaderOp, PixMap, optional ColorTable, and CopyBits opcode. Output is suitable for picture_to_surface().

### calculate_picture_clut
- **Signature:** `struct color_table *calculate_picture_clut(int CLUTSource, int pict_resource_number)`
- **Purpose:** Load a color table resource and convert it to engine format.
- **Inputs:** Source file selector (CLUTSource_Images or CLUTSource_Scenario), resource ID
- **Outputs/Return:** Newly allocated color_table; NULL if not found or not 8-bit display
- **Side effects:** Allocates color_table; converts MacOS CLUT to engine format
- **Calls:** `get_clut()`, `build_color_table()` or `build_direct_color_table()`
- **Notes:** Returns NULL for non-8-bit depths (direct color used instead).

### get_picture_resource_from_images / get_picture_resource_from_scenario
- **Signature:** `bool get_picture_resource_from_images(int base_resource, LoadedResource &PictRsrc)` (and scenario variant)
- **Purpose:** High-level API to retrieve a picture resource with bit-depth fallback.
- **Inputs:** Base resource ID, output LoadedResource
- **Outputs/Return:** `true` if loaded; `false` otherwise
- **Side effects:** Calls determine_pict_resource_id and get_pict
- **Calls:** `determine_pict_resource_id()`, `get_pict()`
- **Notes:** Assertion ensures file is open; scenario version returns false if file not open.

## Control Flow Notes
**Initialization:** `initialize_images_manager()` is called at engine startup via main/shell code, opening ImagesFile and registering shutdown handler.

**Per-scenario:** `set_scenario_images_file()` is called when loading a new scenario/map, opening ScenarioFile.

**Per-frame/on-demand:** Picture retrieval is driven by game state (e.g., intro screens, menu display). `draw_full_screen_pict_resource_from_images()` and `draw_full_screen_pict_resource_from_scenario()` are called to display full-screen images with optional scrolling.

**Shutdown:** `shutdown_images_handler()` (registered with atexit) closes both file objects on process termination.

## External Dependencies
- **FileHandler.h:** `FileSpecifier`, `OpenedFile`, `OpenedResourceFile`, `LoadedResource` (object-oriented file I/O abstraction)
- **wad.h:** WAD file structures and functions (`wad_data`, `read_indexed_wad_from_file()`, `extract_type_from_wad()`, `free_wad()`)
- **screen_drawing.h:** Clipping state (`draw_clip_rect_active`, `draw_clip_rect`); color/shading utilities
- **SDL, SDL_image:** `SDL_RWops`, `SDL_Surface`, `SDL_BlitSurface()`, `IMG_LoadTyped_RW()` (rendering and image decompression)
- **byte_swapping.h:** `byte_swap_memory()` for endian conversion
- **cseries.h, interface.h:** Macros (FOUR_CHARS_TO_INT), constants, memory management
- **screen.h:** `interface_bit_depth` (external global); called by shell for full-screen image display
