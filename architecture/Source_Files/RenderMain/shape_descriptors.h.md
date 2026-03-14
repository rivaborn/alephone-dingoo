# Source_Files/RenderMain/shape_descriptors.h

## File Purpose
Defines the packed `shape_descriptor` bitfield structure and associated utility macros for referencing graphics assets in the game. Shape descriptors encode a collection ID, shape index, and color lookup table (CLUT) into a single 16-bit value to efficiently organize and retrieve sprite/shape graphics used in rendering.

## Core Responsibilities
- Define `shape_descriptor` as a 16-bit packed bitfield type
- Enumerate all 32 asset collections (weapons, monsters, scenery, landscapes, UI, etc.)
- Provide bit-field layout constants (shape: 8 bits, collection: 5 bits, CLUT: 3 bits)
- Offer extraction macros to unpack descriptor components
- Provide construction macros to assemble descriptors from collection/shape pairs

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef | 16-bit packed bitfield: `[CLUT.3][collection.5][shape.8]` |

## Global / File-Static State
None.

## Key Functions / Methods
None. (Pure typedef and macro definitions.)

## Control Flow Notes
This header is a data-definition utility, not part of the main frame loop. It is included by rendering code that needs to look up or construct shape references. The descriptor format allows efficient packing of asset metadata into a single word for memory efficiency and fast access during sprite/shape rendering.

## External Dependencies
- `uint16` type (platform-defined, likely from stdint or engine headers)

## Notes
- The enum `_collection_*` defines 32 asset collection IDs (0ΓÇô31), covering enemies, weapons, scenery, landscapes, and UI.
- Collections are thematically grouped: walls/scenery by environment (water, lava, sewage, Jjaro, Pfhor), landscapes by time-of-day (day, night, moon, space).
- Macros handle bitwise extraction/packing; the naming convention (e.g., `_collection_civilian_fusion` renamed from `_collection_madd`) reflects legacy refactors.
- Header was historically not in the makefile, per commentΓÇöchanges require manual touch of build system.
