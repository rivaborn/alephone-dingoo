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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `tint_table8` | struct | 8-bit indexed color lookup table (palette remapping) |
| `tint_table16` | struct | 16-bit RGB color channel lookup tables for tinting |
| `tint_table32` | struct | 32-bit RGB color channel lookup tables for tinting |
| `rectangle_definition` | struct | Sprite/rectangle rendering object with position, depth, shading, opacity, and 3D model data |
| `polygon_definition` | struct | Textured polygon (wall) rendering object with vertices, origin, shading, and shape descriptor |
| `shape_descriptor` | typedef (uint16) | Packed 16-bit descriptor encoding CLUT (3 bits), collection (5 bits), shape (8 bits) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `bit_depth` | short | extern global | Current rendering color depth (8/16/32-bit) |
| `interface_bit_depth` | short | extern global | UI rendering color depth |
| `number_of_shading_tables` | short | extern global | Count of available shading lookup tables |
| `shading_table_fractional_bits` | short | extern global | Fixed-point precision for shading calculations |
| `shading_table_size` | short | extern global | Size of each shading table in bytes |
| `MAXIMUM_SHADING_TABLE_INDEXES` | macro constant | compile-time | Max shading table indices (= `PIXEL8_MAXIMUM_COLORS`) |
| `FIRST_SHADING_TABLE` | macro constant | compile-time | Base index for shading tables (0) |
| `_SHADELESS_BIT` | macro constant | compile-time | Flag (0x8000) to ignore shading tables |

## Key Functions / Methods

### allocate_texture_tables
- **Signature:** `void allocate_texture_tables(void);`
- **Purpose:** Initialize and allocate memory for shading lookup tables used during rendering.
- **Inputs:** None (uses global state).
- **Outputs/Return:** None.
- **Side effects (global state, I/O, alloc):** Allocates memory for shading tables; modifies global texture state.
- **Calls:** Not visible in this file.
- **Notes:** Called during engine initialization; critical for texture rendering pipeline startup.

## Control Flow Notes
This header is used during the **render phase** of each frame. `rectangle_definition` objects hold sprite/HUD geometry and shading state; `polygon_definition` objects hold wall/surface geometry. Both are populated by geometry/scene preparation and consumed by the rasterizer/OpenGL renderer. Global shading table state is set up at initialization via `allocate_texture_tables()` and referenced during rasterization.

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
