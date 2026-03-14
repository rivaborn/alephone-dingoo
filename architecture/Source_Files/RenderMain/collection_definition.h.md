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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| collection_definition | struct | Root container holding offset tables to color tables, high/low-level shapes, and bitmaps; includes scaling and size metadata |
| high_level_shape_definition | struct | Sprite animation metadata: frame/view counts, tick timing, transfer mode, sound events, and flexible array of low-level shape indices |
| low_level_shape_definition | struct | Individual sprite frame: bitmap index, pixel/world origin and keypoint, bounds, lighting intensity, mirror/obscure flags |
| rgb_color_value | struct | Color palette entry: 8-bit flags and value, plus 16-bit RGB components |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| COLLECTION_VERSION | macro (3) | global | Binary format version for collection file compatibility |
| NUMBER_OF_PRIVATE_COLORS | macro (3) | global | Reserved palette entries at start of color table |
| SIZEOF_collection_definition | const int (544) | global | Binary size validation constant |
| SIZEOF_high_level_shape_definition | const int (90) | global | Binary size validation constant |
| SIZEOF_low_level_shape_definition | const int (36) | global | Binary size validation constant |
| SIZEOF_rgb_color_value | const int (8) | global | Binary size validation constant |

## Key Functions / Methods

NoneΓÇöthis is a pure definition file.

## Control Flow Notes

Used during resource loading and sprite rendering:
1. **Load phase:** Deserialize collection_definition from disk; use offset tables and counts to populate std::vector members (color tables, shapes, bitmaps)
2. **Render phase:** For animated sprites, traverse high_level_shape_definition's index array to fetch current low_level_shape_definition, resolve bitmap, apply transfer mode and lighting
3. **Coordinate mapping:** pixels_to_world field scales pixel coordinates to world space; origin/keypoint define sprite positioning

## External Dependencies

- Standard library: std::vector (for dynamic arrays)
- Assumed custom types: `_fixed` (fixed-point type, defined elsewhere), transfer mode constants (likely from interface.h)
- Forward declarations: bitmap_definition, high_level_shape_definition, low_level_shape_definition (satisfy internal references)

## Notes

- Flags use bitfield notation: `[x-mirror.1] [y-mirror.1] [keypoint_obscured.1]` indicates individual flag bits
- `low_level_shape_indexes[1]` is a flexible array member (actual length determined by animation data)
- `unused` arrays provide binary padding for format evolution
- Color table supports multiple palettes (clut_count arrays of color_count entries)
- Includes old changelog comments (1994) showing incremental format evolution from early development
