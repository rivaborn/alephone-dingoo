# Source_Files/XML/ColorParser.cpp

## File Purpose
An XML element parser for color definitions in the Aleph One game engine. Parses `<color>` XML elements with red, green, and blue float attributes, converts them to 16-bit RGB values, and stores them in a caller-provided color array at a specified or default index.

## Core Responsibilities
- Parse XML `<color>` element attributes (red, green, blue, optional index)
- Validate that all required attributes are present in the element
- Convert floating-point color channel values [0ΓÇô1] to 16-bit unsigned integers
- Clamp color values to valid range [0, 65535]
- Store parsed colors into a caller-supplied `rgb_color` array
- Support both indexed and non-indexed color storage modes
- Provide a singleton parser instance for reuse across multiple color element parses

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_ColorParser | class | Extends `XML_ElementParser`; manages state for parsing a single `<color>` element |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| ColorParser | XML_ColorParser | static | Singleton parser instance; reused for all color element parsing |

## Key Functions / Methods

### Start
- Signature: `bool XML_ColorParser::Start()`
- Purpose: Initialize parser state when a new `<color>` element begins
- Inputs: None (uses `this` context)
- Outputs/Return: `true` (always succeeds)
- Side effects: Clears `IsPresent[0ΓÇô3]` flags to false
- Calls: None
- Notes: Resets the presence flags for red, green, blue, and index attributes

### HandleAttribute
- Signature: `bool XML_ColorParser::HandleAttribute(const char *Tag, const char *Value)`
- Purpose: Parse individual XML attributes (red, green, blue, index) and populate `TempColor`
- Inputs: `Tag` ΓÇô attribute name; `Value` ΓÇô attribute value string
- Outputs/Return: `true` if attribute recognized and parsed successfully; `false` otherwise
- Side effects: Updates `TempColor.red`, `.green`, `.blue` and `Index` member fields; sets `IsPresent` flags
- Calls: `StringsEqual()` (defined elsewhere), `ReadBoundedInt16Value()`, `ReadFloatValue()`, `UnrecognizedTag()` (defined elsewhere)
- Notes: Converts float values [0ΓÇô1] to 16-bit range via `PIN(65535*CVal+0.5, 0, 65535)`; index is only parsed if `NumColors > 0` (non-indexed mode skips index checks); note: `CVal` is redeclared in the blue branch (minor code smell)

### AttributesDone
- Signature: `bool XML_ColorParser::AttributesDone()`
- Purpose: Validate that all required attributes were present, then commit the parsed color to the target array
- Inputs: None (uses member state: `NumColors`, `IsPresent`, `TempColor`, `Index`)
- Outputs/Return: `true` if validation passes and color is stored; `false` if required attributes are missing
- Side effects: Sets `IsPresent[3] = true` and `Index = 0` if in non-indexed mode; writes `TempColor` to `ColorList[Index]` on success
- Calls: `AttribsMissing()` (defined elsewhere)
- Notes: In non-indexed mode (`NumColors <= 0`), index is not required and defaults to 0; asserts that `ColorList` is non-null before indexing

## File-Level Functions

### Color_GetParser
- Signature: `XML_ElementParser *Color_GetParser()`
- Purpose: Return the singleton color parser for use in XML parsing workflows
- Inputs: None
- Outputs/Return: Pointer to static `ColorParser` instance
- Side effects: None
- Notes: Allows multiple XML elements to reuse the same parser instance by resetting it via `Start()` between elements

### Color_SetArray
- Signature: `void Color_SetArray(rgb_color *ColorList, int NumColors)`
- Purpose: Configure the target color array and count for subsequent color parses
- Inputs: `ColorList` ΓÇô pointer to array of `rgb_color`; `NumColors` ΓÇô count of colors (if Γëñ 0, disables indexed mode)
- Outputs/Return: None
- Side effects: Updates `ColorParser.ColorList` and `ColorParser.NumColors` static fields
- Notes: Must be called before parsing begins to set the destination array; passing `NumColors <= 0` switches to non-indexed mode

## Control Flow Notes
Typical XML parser lifecycle:
1. **Init**: `Color_SetArray()` is called once to configure the parser with a target array
2. **Per element**: `Color_GetParser()` returns the singleton; XML framework calls `Start()`, then repeated `HandleAttribute()` calls, then `AttributesDone()`
3. **Repeat**: Parser is reused for each subsequent `<color>` element in the XML tree

## External Dependencies
- **Includes**: `<string.h>`, `"ColorParser.h"`, (transitively: `"XML_ElementParser.h"`, `"cseries.h"`)
- **Helper functions** (defined elsewhere): `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `UnrecognizedTag()`, `AttribsMissing()`
- **Types** (defined elsewhere): `XML_ElementParser` (base class), `rgb_color` (16-bit RGB struct), `uint16`
- **Macros** (defined elsewhere): `PIN()` (likely bounds-clamp: `PIN(val, min, max)`)
