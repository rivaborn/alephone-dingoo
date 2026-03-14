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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `collection_definition` | struct | Loaded collection metadata: colors, shapes, bitmaps, counts/offsets |
| `collection_header` | struct | Runtime collection state: loaded definition, shading/tint tables, status flags |
| `bitmap_definition` | struct | Bitmap metadata: dimensions, row stride, RLE flags, bit depth |
| `high_level_shape_definition` | struct | Animation frame definition: views, frames per view, sounds, transfer mode |
| `low_level_shape_definition` | struct | Individual sprite frame: bitmap index, mirroring flags, world bounds |
| `rgb_color_value` | struct | Color entry: flags (luminescence), palette index, 16-bit RGB channels |
| `XML_InfravisionAssignParser` | class | XML parser for assigning tint colors to specific collections |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `global_shading_table16` | `pixel16*` | static | Pre-computed shading multipliers for 16-bit color components |
| `global_shading_table32` | `pixel32*` | static | Pre-computed shading multipliers for 32-bit color components |
| `ShapesFile` | `OpenedFile` | static | File handle to opened shapes.shp resource |
| `number_of_shading_tables` | `short` | global | Count of darkness/brightness levels (e.g., 32 levels) |
| `shading_table_size` | `short` | global | Bytes per shading table (typically 256 for palette lookup) |
| `CollectionTints[NUMBER_OF_COLLECTIONS]` | `short[]` | global | Per-collection infravision tint color indices |
| `tint_colors16[NUMBER_OF_TINT_COLORS]` | `rgb_color[]` | static | 16-bit RGB values for infravision tints (red, green, blue, yellow) |
| `tint_colors8[NUMBER_OF_TINT_COLORS]` | `tint_color8_data[]` | static | 8-bit palette range assignments for infravision tints |

## Key Functions / Methods

### get_shape_surface
- **Signature:** `SDL_Surface *get_shape_surface(int shape, int inCollection, byte** outPointerToPixelData, float inIllumination, bool inShrinkImage)`
- **Purpose:** Convert a shape descriptor (or separate collection + shape index) into a drawable SDL surface with color palette applied.
- **Inputs:** Shape descriptor, optional explicit collection/CLUT, illumination level (0.0ΓÇô1.0 for shading), shrink flag for RLE shapes
- **Outputs/Return:** SDL_Surface with pixel data and color table; optionally returns malloc'd pixel buffer for RLE shapes (caller must free)
- **Side effects:** Allocates SDL surface and optionally pixel buffers; accesses shading tables and color tables from loaded collections
- **Calls:** `get_collection_definition`, `get_low_level_shape_definition`, `extended_get_shape_bitmap_and_shading_table`, `get_bitmap_definition`, `get_collection_colors`, SDL surface creation/manipulation functions
- **Notes:** Handles RLE decompression inline with row/column-major unpacking and optional mirroring. Shrinking is nearest-neighbor only (no smoothing). Illumination < 0 uses collection CLUT directly; >= 0 indexes shading tables.

### load_collection
- **Signature:** `static bool load_collection(short collection_index, bool strip)`
- **Purpose:** Load a single collection's data from ShapesFile: parse definition, CLUTs, shapes, bitmaps; allocate shading tables.
- **Inputs:** Collection index (0ΓÇô31), strip flag (skip bitmap data if true)
- **Outputs/Return:** Populates `collection_header->collection` and shading/tint table allocations; returns success
- **Side effects:** Seeks in ShapesFile, calls malloc for all buffers, byte-swaps big-endian data, calls `build_collection_tinting_table`
- **Calls:** `load_collection_definition`, `load_clut`, `load_high_level_shape`, `load_low_level_shape`, `load_bitmap`, `byte_swap_memory`, `build_collection_tinting_table`, `allocate_shading_tables`
- **Notes:** Uses `std::auto_ptr` for exception-safe temp storage. RLE-format CLUTs and bitmaps recompute size at read-time. Patches infravision tinting flags.

### build_shading_tables16 / build_shading_tables32
- **Signature:** `static void build_shading_tables16(rgb_color_value *colors, short color_count, pixel16 *shading_tables, byte *remapping_table, bool is_opengl)`
- **Purpose:** Generate lightness-modulated pixel values for each color in each shading level; store in contiguous table for fast lookup.
- **Inputs:** Color array, count, output table buffer, optional color index remapping, OpenGL flag
- **Outputs/Return:** Fills shading_tables[PIXEL8_MAXIMUM_COLORS * level + palette_index] with pixel value at that brightness
- **Side effects:** Calls SDL_MapRGB (SDL mode) or bit-shifts xRGB values (OpenGL mode)
- **Calls:** `get_next_color_run`, `SDL_MapRGB`, bit macros (`RGBCOLOR_TO_PIXEL16`, `RGBCOLOR_TO_PIXEL32`)
- **Notes:** Self-luminescent colors (flag 0x80) apply half-brightness floor. Color runs group monotonic-brightness colors for cache locality.

### build_global_shading_table16 / build_global_shading_table32
- **Signature:** `static void build_global_shading_table16(void)` (no params)
- **Purpose:** Pre-compute per-component darkening curves (one curve per RGB channel, per shading level) for fast pixel-component multiplication.
- **Inputs:** None (reads `number_of_shading_tables`, SDL pixel format if present)
- **Outputs/Return:** Allocates and fills `global_shading_table16/32`
- **Side effects:** malloc; computed once on first call, then cached
- **Calls:** SDL_PixelFormat field access or bit shifts
- **Notes:** Lazy initialization. Under SDL, accounts for variable component widths/shifts. Deferred until first use.

### Infravision XML parsers (XML_InfravisionAssignParser, XML_InfravisionParser)
- **Purpose:** Parse XML elements `<infravision><colors><assign.../></colors></infravision>` to define tint colors and per-collection assignments.
- **Inputs:** XML attributes (coll=index, color=tint_id)
- **Side effects:** Modifies global `CollectionTints[]` array; backs up originals for reset
- **Notes:** Two-level parser: outer handles color definitions (delegates to `Color_GetParser()`), inner assigns them to collections. Supports XML-driven customization without recompile.

---

### Accessor functions (all static, guards against NULL/out-of-range)
- `get_collection_header(short idx)` ΓåÆ `collection_header*` | returns bounds-checked entry from global array
- `get_collection_definition(short idx)` ΓåÆ `collection_definition*` | returns loaded collection or NULL
- `get_collection_colors(short idx, short clut)` ΓåÆ `rgb_color_value*` | returns CLUT pointer with bounds check
- `get_high_level_shape_definition(short coll, short high_idx)` ΓåÆ pointer to animation data or NULL
- `get_low_level_shape_definition(short coll, short low_idx)` ΓåÆ pointer to frame data or NULL
- `get_bitmap_definition(short coll, short bitmap_idx)` ΓåÆ pointer to bitmap or NULL
- `get_collection_shading_tables(short coll, short clut)` ΓåÆ void* | byte-offsets shading table buffer
- `get_collection_tint_tables(short coll, short tint_idx)` ΓåÆ void* | byte-offsets tint table buffer

**Notes on accessors:** All perform null-checks and bounds validation; return NULL on failure to allow graceful degradation. No assertions on public-facing queries (user-friendly).

---

## Control Flow Notes

- **Initialization:** `initialize_shape_handler()` stub; XML infravision parser registered via `Infravision_GetParser()`
- **Load phase:** `open_shapes_file()` opens file; `load_collections()` iterates marked collections, calls `load_collection()` per index
- **Render phase:** Rendering code calls `get_shape_surface()` or `extended_get_shape_bitmap_and_shading_table()` to fetch drawable data
- **Per-frame:** Shading tables may be indexed by illumination (lighting effects); tint tables applied for infravision mode
- **Shutdown:** `shutdown_shape_handler()` would free global tables and close file

Not fully visible: tint tables and shading tables are built during `load_collection()` and indexed later during rendering.

## External Dependencies

- **SDL:** `SDL_RWops`, `SDL_ReadBE16/32`, `SDL_CreateRGBSurfaceFrom`, `SDL_SetColors`, `SDL_SetColorKey`, `SDL_PixelFormat`, `SDL_MapRGB`, `SDL_SwapBE16`
- **File I/O:** `OpenedFile`, `FileSpecifier` (from FileHandler.h)
- **Rendering:** `OGL_Render.h`, `OGL_LoadScreen.h` (conditional OpenGL support)
- **XML parsing:** `XML_ElementParser`, `ColorParser.h`, `Color_GetParser()`, `Color_SetArray()`
- **Data structures:** `collection_definition`, `bitmap_definition`, `rgb_color_value` (from collection_definition.h)
- **Utilities:** `byte_swapping.h`, `Packing.h`, `SW_Texture_Extras.h`, `progress.h`, `cspixels.h` (color macros)
- **Defined elsewhere:** `bit_depth` (global), `objlist_set()`, `vassert()`, `csprintf()`, shape descriptor macros (`GET_COLLECTION`, `GET_DESCRIPTOR_*`)
