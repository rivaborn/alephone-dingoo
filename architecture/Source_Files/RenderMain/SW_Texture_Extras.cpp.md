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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SW_Texture` | class | Single texture with opacity parameters, descriptor, and opacity lookup table |
| `SW_Texture_Extras` | class | Singleton container for all textures, indexed by collection; manages Load/Unload |
| `XML_SW_Texture_Parser` | class | XML parser for individual `<texture>` elements with opacity attributes |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SW_Texture_Extras::m_instance` | `SW_Texture_Extras*` | static | Singleton instance for global texture management |
| `SW_Texture_Parser` | `XML_SW_Texture_Parser` | static | Global parser for texture XML elements |
| `SW_Texture_Extras_Parser` | `XML_ElementParser` | static | Global parser for software-textures XML wrapper |
| `bit_depth` | `short` (extern) | external | Current display bit depth (16 or 32); affects opacity table format |

## Key Functions / Methods

### SW_Texture::build_opac_table
- **Signature:** `void build_opac_table()`
- **Purpose:** Construct opacity lookup table by sampling the shading table at the darkest index and mapping color intensity to opacity value.
- **Inputs:** None (uses member `m_shape_descriptor`, `m_opac_type`, `m_opac_scale`, `m_opac_shift`)
- **Outputs/Return:** Populates `m_opac_table` (vector of uint8, indexed by shading table index)
- **Side effects:** Resizes and clears `m_opac_table`; retrieves bitmap and shading tables via `get_shape_bitmap_and_shading_table`
- **Calls:** `get_shape_bitmap_and_shading_table()`, `SDL_GetVideoSurface()`, `PIN()` macro (clamp)
- **Notes:** Branches on `bit_depth` (16 vs 32); three opacity modes: (1) fully opaque, (2) average RGB, (3) max RGB channel. Uses `m_opac_scale * opacity + 255 * m_opac_shift` formula with clamping to [0, 255].

### SW_Texture_Extras::AddTexture
- **Signature:** `SW_Texture *AddTexture(shape_descriptor ShapeDesc)`
- **Purpose:** Retrieve or create a texture for a given shape descriptor, resizing collection vector as needed.
- **Inputs:** `ShapeDesc` (shape descriptor with collection and bitmap indices)
- **Outputs/Return:** Pointer to `SW_Texture` in the collection array
- **Side effects:** May resize `texture_list[Collection]` vector
- **Calls:** `GET_COLLECTION()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_DESCRIPTOR_SHAPE()`
- **Notes:** Does not call `build_opac_table()`; that is deferred to `Load()`.

### SW_Texture_Extras::GetTexture
- **Signature:** `SW_Texture *GetTexture(shape_descriptor ShapeDesc)`
- **Purpose:** Safely retrieve a texture without creating if absent.
- **Inputs:** `ShapeDesc`
- **Outputs/Return:** Pointer to `SW_Texture`, or null if bitmap index is out of range
- **Side effects:** None
- **Calls:** `GET_COLLECTION()`, `GET_DESCRIPTOR_COLLECTION()`, `GET_DESCRIPTOR_SHAPE()`

### SW_Texture_Extras::Load
- **Signature:** `void Load(short collection_index)`
- **Purpose:** Build opacity tables for all textures in a collection (called when collection is loaded into memory).
- **Inputs:** `collection_index`
- **Outputs/Return:** None
- **Side effects:** Iterates collection and calls `build_opac_table()` on each texture
- **Calls:** Iterator loop over `texture_list[collection_index]`

### SW_Texture_Extras::Unload
- **Signature:** `void Unload(short collection_index)`
- **Purpose:** Release opacity table memory for all textures in a collection.
- **Inputs:** `collection_index`
- **Outputs/Return:** None
- **Side effects:** Calls `clear_opac_table()` on each texture (vector clear)
- **Calls:** Iterator loop over `texture_list[collection_index]`

### XML_SW_Texture_Parser::Start
- **Signature:** `bool Start()`
- **Purpose:** Initialize parser state before parsing a texture element.
- **Outputs/Return:** `true`
- **Side effects:** Sets `OpacityType = 0`, `OpacityScale = 1.0`, `OpacityShift = 0.0`

### XML_SW_Texture_Parser::HandleAttribute
- **Signature:** `bool HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Parse individual XML attributes (`coll`, `bitmap`, `opac_type`, `opac_scale`, `opac_shift`).
- **Inputs:** XML tag name and value string
- **Outputs/Return:** `true` on recognized/valid attribute, `false` on error
- **Calls:** `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `UnrecognizedTag()`
- **Notes:** Collection and bitmap are bounded; opacity type is 0ΓÇô3.

### XML_SW_Texture_Parser::AttributesDone
- **Signature:** `bool AttributesDone()`
- **Purpose:** Finalize texture after all attributes are parsed; add to manager.
- **Outputs/Return:** `true`
- **Side effects:** Calls `SW_Texture_Extras::instance()->AddTexture()` and sets opacity parameters
- **Calls:** `BUILD_DESCRIPTOR()`, singleton accessor

### SW_Texture_GetParser / SW_Texture_Extras_GetParser
- **Purpose:** Export parser objects to caller (e.g., XML configuration loader)
- **Outputs/Return:** Pointer to static parser instance

## Control Flow Notes
- **Initialization:** XML parsers are created at module load; singleton instance is created on first `instance()` call.
- **Collection Loading:** When a collection is loaded (e.g., during level init), `Load(collection_index)` is called, triggering `build_opac_table()` for all textures in that collection.
- **Collection Unloading:** On unload, `Unload()` releases opacity table memory to free graphics memory.
- **Runtime:** Opacity tables are indexed by shading table index during software blitting (not shown in this file); enables fast per-pixel opacity lookup.

## External Dependencies
- **`interface.h`** ΓÇô `get_shape_bitmap_and_shading_table()`, shape/collection accessors, macros (`GET_DESCRIPTOR_COLLECTION`, `GET_DESCRIPTOR_SHAPE`, `BUILD_DESCRIPTOR`)
- **`collection_definition.h`** ΓÇô `NUMBER_OF_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION` constants
- **`render.h`** ΓÇô `number_of_shading_tables`, `MAXIMUM_SHADING_TABLE_INDEXES`
- **`scottish_textures.h`** ΓÇô Transfer mode and shading table definitions
- **SDL** ΓÇô `SDL_PixelFormat`, `SDL_GetVideoSurface()` for pixel format introspection
- **`XML_ElementParser.h`** ΓÇô XML parser base class
- **Standard library** ΓÇô `<vector>`, `std::max()`
- **`bit_depth`** (extern) ΓÇô Global color bit depth indicator
