# Source_Files/RenderMain/SW_Texture_Extras.h

## File Purpose
Defines classes for managing software-rendered texture properties, including opacity tables and scaling/shifting parameters. Part of the Aleph One game engine's rendering subsystem, providing XML-loadable texture metadata organized by game asset collections.

## Core Responsibilities
- Encapsulate per-texture opacity properties (type, scale, shift, lookup table)
- Provide singleton access to texture metadata keyed by shape descriptor
- Manage texture loading/unloading per collection
- Support XML-based serialization of texture configurations
- (Conditionally compiled: disabled on Dingoo platform for binary size)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SW_Texture` | class | Wraps a single texture's opacity and rendering properties |
| `SW_Texture_Extras` | class | Singleton container for all textures, indexed by collection |
| `shape_descriptor` | typedef (uint16) | Identifies a texture by collection, shape, and CLUT (16-bit packed) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SW_Texture_Extras::m_instance` | `SW_Texture_Extras*` | static | Singleton instance pointer |

## Key Functions / Methods

### SW_Texture::descriptor
- **Signature:** `void descriptor(shape_descriptor ShapeDesc)`
- **Purpose:** Set the shape descriptor associated with this texture
- **Inputs:** `ShapeDesc` ΓÇô packed shape/collection/CLUT identifier
- **Outputs/Return:** None
- **Side effects:** Modifies `m_shape_descriptor`

### SW_Texture::opac_table
- **Signature:** `uint8 *opac_table()`
- **Purpose:** Retrieve pointer to opacity lookup table data
- **Inputs:** None
- **Outputs/Return:** Pointer to first element of `m_opac_table`, or null if empty
- **Side effects:** None
- **Notes:** Returns dangling pointer if vector reallocates after call; assumes external code does not hold reference across modifications

### SW_Texture::build_opac_table
- **Signature:** `void build_opac_table()`
- **Purpose:** Populate the opacity table based on current scale/shift parameters
- **Inputs:** None (uses member state)
- **Outputs/Return:** None
- **Side effects:** Modifies `m_opac_table`
- **Notes:** Implementation not in this header; likely applies scale/shift to generate final lookup values

### SW_Texture_Extras::instance
- **Signature:** `static SW_Texture_Extras *instance()`
- **Purpose:** Lazy-initialization singleton accessor
- **Inputs:** None
- **Outputs/Return:** Pointer to singleton instance
- **Side effects:** Allocates singleton on first call (heap allocation via `new`)
- **Notes:** Not thread-safe; no destructor defined (singleton lives for process lifetime)

### SW_Texture_Extras::GetTexture
- **Signature:** `SW_Texture *GetTexture(shape_descriptor ShapeDesc)`
- **Purpose:** Retrieve texture metadata by shape descriptor
- **Inputs:** `ShapeDesc` ΓÇô packed identifier
- **Outputs/Return:** Pointer to matching `SW_Texture`, or null if not found
- **Side effects:** None

### SW_Texture_Extras::AddTexture
- **Signature:** `SW_Texture *AddTexture(shape_descriptor ShapeDesc)`
- **Purpose:** Create and register a new texture entry
- **Inputs:** `ShapeDesc` ΓÇô identifier for the new texture
- **Outputs/Return:** Pointer to newly created `SW_Texture`
- **Side effects:** Modifies appropriate `texture_list[collection]`

### SW_Texture_Extras::Load / Unload
- **Signature:** `void Load(short Collection)`, `void Unload(short Collection)`
- **Purpose:** Load/unload texture data for a specific collection (e.g., walls, scenery, enemies)
- **Inputs:** `Collection` ΓÇô collection index (0ΓÇô31, per `shape_descriptors.h`)
- **Outputs/Return:** None
- **Side effects:** Populate or clear `texture_list[Collection]` (likely from XML or asset files)
- **Notes:** Implementation details not in this header

### SW_Texture_Extras_GetParser
- **Signature:** `XML_ElementParser *SW_Texture_Extras_GetParser()`
- **Purpose:** Factory function returning an XML element parser for texture configuration
- **Inputs:** None
- **Outputs/Return:** Pointer to new/existing `XML_ElementParser` subclass
- **Side effects:** Likely allocates parser; ownership/lifetime unclear from this header
- **Notes:** Implementation not in this header; expected to parse XML elements into texture properties

## Control Flow Notes
Fits into resource/asset initialization pipeline: `Load(collection)` is called at level load or initialization time, populating texture metadata from XML or binary asset files. Opacity tables are built on-demand via `build_opac_table()` during rendering setup. The singleton pattern suggests engine-wide, centralized access during frame rendering.

## External Dependencies
- **Includes:** `config.h`, `cseries.h`, `cstypes.h`, `shape_descriptors.h`, `XML_ElementParser.h`, `<vector>`
- **Types/Constants defined elsewhere:**
  - `shape_descriptor`, `NUMBER_OF_COLLECTIONS`, collection enum constants (from `shape_descriptors.h`)
  - `uint8`, `int` (from `cstypes.h`)
  - `XML_ElementParser` base class (from `XML_ElementParser.h`)
- **Conditional compilation:** Entire file omitted if `HAVE_DINGOO` is defined (Dingoo platform optimization)
