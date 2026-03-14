# Source_Files/RenderMain/shape_definitions.h

## File Purpose
Header file defining core data structures for shape/collection management in the rendering system. Manages metadata about graphics collections (sprites, textures, shading tables) that are loaded from disk and used during rendering.

## Core Responsibilities
- Defines the `collection_header` struct that describes an in-memory graphics collection
- Tracks disk offsets and memory layouts for collections (standard and 16-bit variants)
- Maintains a global array of collection headers indexed during rendering
- Provides constants for data structure sizes and array bounds

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `collection_header` | struct | Metadata for a single collection: status/flags, disk offsets, pointers to in-memory collection and shading tables |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `collection_headers` | `collection_header[MAXIMUM_COLLECTIONS]` | static | Global registry of all loaded collections; indexed during rendering/shape lookup |
| `SIZEOF_collection_header` | const int | static | Size constant (32 bytes on disk) for serialization/deserialization |

## Key Functions / Methods
None. This is a data structure definition header.

## Notes
- Trivial helper: `SIZEOF_collection_header` constant documents disk layout size (32 bytes).
- Collection and shading table pointers were converted from MacOS-specific handles to standard pointers (Aug 2000 refactor) to support variable-format objects.
- `status` and `flags` fields suggest collections may be conditionally loaded or have runtime state.
- Dual offset/length fields (`offset`/`length` and `offset16`/`length16`) indicate support for standard and 16-bit variants, likely for compressed or alternate texture formats.

## Control Flow Notes
Part of the **initialization/asset loading** pipeline. Collections are loaded from disk (using offset/length fields) and registered in the global `collection_headers` array. During **rendering**, shapes are looked up by collection index and retrieved from the registered pointers. Shading tables are applied during lighting/color calculations.

## External Dependencies
- `collection_definition` (defined elsewhere; pointer to in-memory collection object)
- Standard C integer types: `int16`, `uint16`, `int32`, `byte`
- Macro: `MAXIMUM_COLLECTIONS` (upper bound for collection registry)
