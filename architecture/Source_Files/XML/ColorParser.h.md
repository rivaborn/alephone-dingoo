# Source_Files/XML/ColorParser.h

## File Purpose
Header file declaring an XML parser interface for color elements in game configuration files. Provides factory and configuration functions for parsing color data (RGB values) into arrays, supporting both indexed and non-indexed color storage.

## Core Responsibilities
- Factory function to instantiate a color XML element parser
- Configuration of target color array for parsed values
- Support for optional indexed color array storage (index attribute only when NumColors > 0)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` | typedef (from cseries.h) | RGB color triplet storage (red, green, blue components) |
| `XML_ElementParser` | class (from XML_ElementParser.h) | Base class for XML element parsing strategy |

## Global / File-Static State
None (header only; implementation file would contain module state).

## Key Functions / Methods

### Color_GetParser
- Signature: `XML_ElementParser *Color_GetParser()`
- Purpose: Factory function to obtain a color element parser instance
- Inputs: None
- Outputs/Return: Pointer to a new or reused `XML_ElementParser` configured for color elements
- Side effects: May allocate parser object
- Calls: Not inferable from header
- Notes: Comment states function is callable multiple times (supports multiple color element parsing contexts)

### Color_SetArray
- Signature: `void Color_SetArray(rgb_color *ColorList, int NumColors = 0)`
- Purpose: Configure the target array into which parsed color values are written
- Inputs: 
  - `ColorList`: Pointer to destination color array
  - `NumColors`: Array size (default 0 = non-indexed mode)
- Outputs/Return: None (void)
- Side effects: Updates internal module state to redirect parser output
- Calls: Not inferable from header
- Notes: When `NumColors <= 0`, parser treats color as non-indexed and ignores "index" XML attributes; when `NumColors > 0`, parser expects and processes index attributes

## Control Flow Notes
Typical usage: call `Color_SetArray()` to configure target, then `Color_GetParser()` to retrieve a parser for XML parsing infrastructure. Parser populates the configured array during XML document traversal.

## External Dependencies
- **Include**: `cseries.h` (core types, SDK)
- **Include**: `XML_ElementParser.h` (parser base class)
- **Symbols used**: `rgb_color` type, `XML_ElementParser` class (both defined elsewhere)
