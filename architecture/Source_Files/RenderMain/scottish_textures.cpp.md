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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `_horizontal_polygon_line_header` | struct | Header for horizontal polygon precalc data (Y downshift) |
| `_horizontal_polygon_line_data` | struct | Per-line texture source coordinates and deltas for horizontal polys |
| `_vertical_polygon_data` | struct | Header for vertical polygon precalc data (width, downshift) |
| `_vertical_polygon_line_data` | struct | Per-line texture pointer, Y coordinates, and shading table for vertical polys |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `scratch_table0`, `scratch_table1` | `short*` | static | Pre-allocated scratch tables for coordinate interpolation; reused across frames |
| `precalculation_table` | `void*` | static | Scratch buffer storing `_vertical_polygon_line_data` or `_horizontal_polygon_line_data` during precalc |
| `texture_random_seed` | `uint16` | static | RNG state for static/randomize transfer mode; persists across frames |

## Key Functions / Methods

### texture_horizontal_polygon
- **Signature**: `void Rasterizer_SW_Class::texture_horizontal_polygon(polygon_definition& textured_polygon)`
- **Purpose**: Main entry point for rendering textured horizontal polygons (walls mapped top-to-bottom). Validates polygon, builds left/right edge coordinate tables via DDA, precalculates per-line texture/shading data, dispatches to template renderers based on bit depth and transfer mode.
- **Inputs**: Reference to `polygon_definition` (vertices, texture, transfer mode, shading mode)
- **Outputs/Return**: None (renders directly to screen buffer)
- **Side effects**: Modifies `scratch_table0`, `scratch_table1`, `precalculation_table`; calls precalc and render functions; may return early if polygon invalid or out of bounds
- **Calls**: `build_x_table()`, `_pretexture_horizontal_polygon_lines()`, `_prelandscape_horizontal_polygon_lines()`, template renderers `texture_horizontal_polygon_lines<>()`, `landscape_horizontal_polygon_lines<>()`
- **Notes**: Punts to `texture_vertical_polygon()` for `_static_transfer` mode (limitations of wall mapper). Handles polygon clipping against screen bounds. Supports 3 transfer modes: `_textured_transfer`, `_static_transfer`, `_big_landscaped_transfer`.

### texture_vertical_polygon
- **Signature**: `void Rasterizer_SW_Class::texture_vertical_polygon(polygon_definition& textured_polygon)`
- **Purpose**: Main entry point for rendering textured vertical polygons (sprites, cliffs mapped left-to-right). Mirrors `texture_horizontal_polygon()` but operates on X axis instead of Y. Swaps to horizontal handler for `_big_landscaped_transfer`.
- **Inputs**: Reference to `polygon_definition`
- **Outputs/Return**: None (renders directly to screen buffer)
- **Side effects**: Same as `texture_horizontal_polygon()` but for vertical edge processing
- **Calls**: `build_y_table()`, `_pretexture_vertical_polygon_lines()`, template renderers `texture_vertical_polygon_lines<>()`
- **Notes**: Only supports `_textured_transfer` and `_static_transfer` modes directly. Delegates landscape to `texture_horizontal_polygon()`.

### texture_rectangle
- **Signature**: `void Rasterizer_SW_Class::texture_rectangle(rectangle_definition& textured_rectangle)`
- **Purpose**: Render axis-aligned textured rectangles with per-column texture lookups and Y range clipping. Builds per-column shading tables and texture Y coordinate stepping.
- **Inputs**: Reference to `rectangle_definition` (bounds, texture, clipping, flip flags, transfer mode)
- **Outputs/Return**: None (renders directly to screen)
- **Side effects**: Computes texture_dx, texture_dy, and fills precalculation table per column; calls template renderers
- **Calls**: `calculate_shading_table()`, template renderers `texture_vertical_polygon_lines<>()`, `randomize_vertical_polygon_lines<>()`, `tint_vertical_polygon_lines<>()`
- **Notes**: Handles horizontal mirroring, clipping (left, right, top, bottom). Supports `_textured_transfer`, `_tinted_transfer`, `_static_transfer` modes. Optimized for repeated texture sampling per screen column.

### _pretexture_horizontal_polygon_lines
- **Signature**: `static void _pretexture_horizontal_polygon_lines(..., struct _horizontal_polygon_line_data *data, short y0, short *x0_table, short *x1_table, short line_count)`
- **Purpose**: Pre-compute texture source coordinates (x, y, dx, dy) and shading tables for each horizontal scan line. Converts world polygon geometry to texture space using view transform matrices.
- **Inputs**: Polygon, view, edge tables (x0_table, x1_table), starting Y, line count
- **Outputs/Return**: Fills `data` array with per-line precalculated values
- **Side effects**: None (pure precalculation)
- **Calls**: `calculate_shading_table()`
- **Notes**: Uses trigonometric lookup tables (`cosine_table`, `sine_table`) and complex fixed-point math for perspective correction. Distinguishes high-precision path when polygon origin Z is near world scale.

### _pretexture_vertical_polygon_lines
- **Signature**: `static void _pretexture_vertical_polygon_lines(..., struct _vertical_polygon_data *data, short x0, short *y0_table, short *y1_table, short line_count)`
- **Purpose**: Pre-compute texture Y coordinates and deltas for each vertical column. Handles division-by-zero safeguards and precision scaling for long-distance camera positions.
- **Inputs**: Polygon, view, edge tables (y0_table, y1_table), starting X, line count
- **Outputs/Return**: Fills `data` array (header + per-line `_vertical_polygon_line_data` records)
- **Side effects**: None (pure precalculation)
- **Calls**: `calculate_shading_table()`
- **Notes**: Complex perspective math involving world-to-screen transforms. Includes multiple scaling passes to avoid overflow. LP modifications for long-distance friendliness.

### _prelandscape_horizontal_polygon_lines
- **Signature**: `static void _prelandscape_horizontal_polygon_lines(..., struct _horizontal_polygon_line_data *data, ...)`
- **Purpose**: Pre-compute landscape texture coordinates using specialized landscape mapping with aspect ratio and repeat options.
- **Inputs**: Polygon, view, scan-line tables, landscape options
- **Outputs/Return**: Fills `data` with landscape-specific texture coordinates
- **Side effects**: Queries landscape options from `View_GetLandscapeOptions()`
- **Calls**: `View_GetLandscapeOptions()`
- **Notes**: Uses separate horizontal/vertical pixel deltas. Supports optional vertical repeat wrapping. Calculates repeat texture height based on OpenGL aspect ratio exponent.

### build_x_table / build_y_table
- **Signature**: `static short *build_x_table(short *table, short x0, short y0, short x1, short y1)` (and Y variant)
- **Purpose**: Bresenham/DDA line rasterization to build coordinate interpolation tables. `build_x_table` interpolates X values for each Y step; `build_y_table` interpolates Y for each X step.
- **Inputs**: Output table pointer, start/end coordinates
- **Outputs/Return**: Pointer to end of populated table (or NULL on invalid input)
- **Side effects**: Populates table in place
- **Calls**: None (pure line rasterization)
- **Notes**: Return NULL if dy < 0 (build_x_table) or dx < 0 (build_y_table) to signal invalid edge. Handles both axis-dominant and near-diagonal cases.

### allocate_texture_tables
- **Signature**: `void allocate_texture_tables(void)`
- **Purpose**: Pre-allocate scratch tables at engine startup.
- **Inputs**: None
- **Outputs/Return**: None (initializes globals)
- **Side effects**: Allocates `scratch_table0`, `scratch_table1`, `precalculation_table`; asserts on failure
- **Calls**: `new` (C++ allocation)
- **Notes**: Called once at startup. Table sizes are constants: `MAXIMUM_SCRATCH_TABLE_ENTRIES = 2048`.

### calculate_shading_table (inline helper)
- **Signature**: `static void calculate_shading_table(void * &result, view_data *view, void *shading_tables, short depth, _fixed ambient_shade)`
- **Purpose**: Select the appropriate pre-baked shading table index based on depth and ambient lighting. Interpolates between ambient shade and depth-based shade.
- **Inputs**: View, depth, ambient shade, shading tables pool
- **Outputs/Return**: Result reference set to pointer into shading_tables
- **Side effects**: Reads `bit_depth`, `number_of_shading_tables` globals
- **Calls**: None
- **Notes**: Macro-like inline to avoid function call overhead. Supports 8, 16, 32-bit output formats.

## Control Flow Notes

**Initialization phase** (engine startup):
- `allocate_texture_tables()` pre-allocates scratch memory

**Per-frame polygon rendering** (render loop):
1. Application calls `texture_horizontal_polygon()` or `texture_vertical_polygon()` for each polygon
2. Validate polygon (bounds, vertex count)
3. **Precalculation phase**: Build edge coordinate tables (`build_x_table` / `build_y_table`), call `_pretexture_*_polygon_lines()` to fill shading and texture data
4. **Rendering phase**: Dispatch to template instantiations (`texture_horizontal_polygon_lines<>()`, etc.) based on bit depth and alpha mode
5. Template functions iterate over pixels, sample textures, apply shading, write to frame buffer

**Rectangle rendering** (special case):
1. `texture_rectangle()` handles per-column texture mapping
2. Builds Y range tables and precalculates texture Y stepping per column
3. Calls vertical polygon renderers in bulk-column mode

## External Dependencies

- `cseries.h` ΓÇô Core series headers (types, macros, assertions)
- `render.h` ΓÇô Rendering structures (`view_data`, `polygon_definition`, `rectangle_definition`, `bitmap_definition`, `point2d`); constants (`MAXIMUM_VERTICES_PER_SCREEN_POLYGON`, `MINIMUM_VERTICES_PER_SCREEN_POLYGON`)
- `Rasterizer_SW.h` ΓÇô Software rasterizer class definition
- `preferences.h` ΓÇô Graphics preferences (`graphics_preferences` global, `software_alpha_blending` mode enum)
- `SW_Texture_Extras.h` ΓÇô Software texture alpha/opacity table support
- `low_level_textures.h` ΓÇô Template implementations for actual pixel writing (`texture_horizontal_polygon_lines<>()`, `landscape_horizontal_polygon_lines<>()`, `texture_vertical_polygon_lines<>()`, `tint_vertical_polygon_lines<>()`, `randomize_vertical_polygon_lines<>()`)
- **Defined elsewhere**: `view_data`, polygon/rectangle definitions, bitmap row addresses, trigonometric lookup tables (`cosine_table`, `sine_table`), `View_GetLandscapeOptions()`, `SDL_Surface`, shading/tint table structures
