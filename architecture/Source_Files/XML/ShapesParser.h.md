# Source_Files/XML/ShapesParser.h

## File Purpose
Header file providing an XML parser factory and configuration interface for parsing shape elements from XML configuration files. Enables multiple parser instances to populate shape_descriptor values with optional "NONE" value support.

## Core Responsibilities
- Provide a factory function (`Shape_GetParser()`) to obtain XML element parsers for shape data
- Configure target pointers where parsed shape descriptors should be written
- Enforce optional "NONE" acceptance policy per parser instance
- Support multiple concurrent parser instances for different shape elements

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef | 16-bit opaque shape reference: packed [clut.3 bits][collection.5 bits][shape.8 bits] |
| `XML_ElementParser` | class (external) | Base class for XML element parsing with attribute/string handling |

## Global / File-Static State
None (state management is delegated to implementation file and the returned parser instances).

## Key Functions / Methods

### Shape_GetParser
- **Signature:** `XML_ElementParser *Shape_GetParser()`
- **Purpose:** Factory function to obtain a new or reusable XML parser instance for shape elements
- **Inputs:** None
- **Outputs/Return:** Pointer to an `XML_ElementParser` configured for shape element parsing
- **Side effects:** May allocate/initialize parser state
- **Calls:** Not visible in header
- **Notes:** Comment emphasizes this should be callable multiple times; suggests reuse or factory pattern

### Shape_SetPointer
- **Signature:** `void Shape_SetPointer(shape_descriptor *DescPtr, bool NONE_Is_OK = true)`
- **Purpose:** Configure the destination pointer and validation rules for the next shape parsing operation
- **Inputs:** 
  - `DescPtr`: target address for parsed shape descriptor
  - `NONE_Is_OK`: whether XML value "NONE" should be treated as valid (default: true)
- **Outputs/Return:** None (void)
- **Side effects:** Modifies internal parser state to point to `DescPtr`; sets acceptance rules for null/none values
- **Calls:** Not visible in header
- **Notes:** Default parameter allows flexibility; NONE_Is_OK likely maps to a sentinel value (e.g., 0xFFFF or special descriptor)

## Control Flow Notes
Part of an XML configuration loading system. Typical flow:
1. Get parser via `Shape_GetParser()`
2. Set output pointer/rules via `Shape_SetPointer()`
3. XML parser (inherited from `XML_ElementParser`) parses element and attributes, writing to configured descriptor pointer
4. Multiple shape elements can be parsed by calling these functions for each shape in sequence

## External Dependencies
- `#include "shape_descriptors.h"` ΓÇô Defines `shape_descriptor` typedef and bitfield macros (`GET_DESCRIPTOR_SHAPE`, `BUILD_DESCRIPTOR`, etc.)
- `#include "XML_ElementParser.h"` ΓÇô Defines `XML_ElementParser` base class with virtual parsing hooks (Start, End, HandleAttribute, HandleString, ResetValues)
- Aleph One game engine framework (Bungie Studios GPL project)
