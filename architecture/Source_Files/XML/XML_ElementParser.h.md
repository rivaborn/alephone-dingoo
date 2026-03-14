# Source_Files/XML/XML_ElementParser.h

## File Purpose
Defines a base class for hierarchical XML element parsing and provides utility functions for parsing numerical, boolean, and string values. Subclasses implement parsing logic for specific XML element types in the Aleph One game engine.

## Core Responsibilities
- Base class (`XML_ElementParser`) for implementing XML element parsers via inheritance
- Template methods for reading and validating typed numerical values (with optional bounds checking)
- Convenience wrappers for common integer, unsigned, float, and boolean value parsing
- ParentΓÇôchild hierarchy management for nested XML elements
- Virtual hooks for element lifecycle (Start/End), attribute handling, string data, and value reset
- Error reporting utilities for missing/invalid attributes, numerical/boolean parsing failures, and out-of-range values
- Global utility functions for case-insensitive string comparison, UTF-8 to ASCII conversion

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_ElementParser` | class | Base class for subclasses implementing element-specific parsing logic |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `XML_GetBooleanValue` | function (extern) | global | Parses a string into a boolean value; defined elsewhere |
| `StringsEqual` | function (extern) | global | Case-insensitive string comparison for XML names/attributes |
| `DeUTF8` | function (extern) | global | Converts UTF-8 to plain ASCII |
| `DeUTF8_Pas`, `DeUTF8_C` | functions (extern) | global | UTF-8 conversion variants for Pascal and C strings |

## Key Functions / Methods

### ReadNumericalValue (template)
- **Signature:** `template<class T> bool ReadNumericalValue(const char *String, const char *Format, T& Value)`
- **Purpose:** Parse a string into a typed numerical value using `sscanf`.
- **Inputs:** String to parse, `sscanf` format string, reference to output value.
- **Outputs/Return:** `true` if parse succeeds; `false` if `sscanf` returns != 1.
- **Side effects:** Calls `BadNumericalValue()` on parse failure.
- **Calls:** `sscanf`, `BadNumericalValue`.
- **Notes:** Used by all typed value readers; triggers error callback on failure.

### ReadBoundedNumericalValue (template)
- **Signature:** `template<class T> bool ReadBoundedNumericalValue(const char *String, const char *Format, T& Value, const T& MinVal, const T& MaxVal)`
- **Purpose:** Parse a string into a numerical value and validate it falls within [MinVal, MaxVal].
- **Inputs:** String, format, value reference, min/max bounds.
- **Outputs/Return:** `true` if parse succeeds and value is in range; `false` otherwise.
- **Side effects:** Calls `OutOfRange()` if value exceeds bounds.
- **Calls:** `ReadNumericalValue`, `OutOfRange`.
- **Notes:** Used for index/identifier parsing with range safety.

### ReadBooleanValue (template)
- **Signature:** `template<class T> bool ReadBooleanValue(const char *String, T& Value)`
- **Purpose:** Parse a string into a boolean and cast it to type `T`.
- **Inputs:** String to parse, reference to output value.
- **Outputs/Return:** `true` if parse succeeds; `false` otherwise.
- **Side effects:** Calls `BadBooleanValue()` on failure.
- **Calls:** `XML_GetBooleanValue`, `BadBooleanValue`.

### ReadInt16Value, ReadUInt16Value, ReadInt32Value, ReadUInt32Value
- **Purpose:** Convenience wrappers for reading signed/unsigned 16-bit and 32-bit integers.
- **Signature:** `bool Read[Signed|Unsigned][Int16|Int32]Value(const char *String, [u]int[16|32]& Value)`
- **Inputs:** String to parse, reference to output value.
- **Outputs/Return:** `true` if successful; `false` otherwise.
- **Calls:** `ReadNumericalValue` with appropriate format string.

### ReadBoundedInt16Value, etc. (bounded variants)
- **Purpose:** Read typed integers with range validation.
- **Signature:** `bool ReadBounded[Type]Value(const char *String, [Type]& Value, [Type] MinVal, [Type] MaxVal)`
- **Calls:** `ReadBoundedNumericalValue` with appropriate format string.

### ReadBooleanValueAs[Type]
- **Purpose:** Read a boolean and cast to int16, uint16, int32, uint32, or bool.
- **Calls:** `ReadBooleanValue` template.

### ReadFloatValue
- **Signature:** `bool ReadFloatValue(const char *String, float& Value)`
- **Purpose:** Parse a string into a float value.
- **Calls:** `ReadNumericalValue` with `"%f"` format.

### Virtual hooks (Start, End, HandleAttribute, AttributesDone, HandleString, ResetValues)
- **Purpose:** Subclasses override these to implement element-specific parsing logic.
- **Signature:** `virtual bool [method](...)`
- **Default behavior:** Return `true` (no-op).
- **Notes:** Return `false` to signal parse error; caller should populate `ErrorString`.

### GetName, NameMatch, AddChild, FindChild, ResetChildrenValues
- **Purpose:** Element hierarchy management and name-based lookup.
- **Notes:** Children are stored in a `vector<XML_ElementParser *>`; `AddChild` prevents duplicate child names (per comments).

## Control Flow Notes
Inferred to be part of XML configuration loading during engine initialization. The hierarchical parentΓÇôchild structure suggests parsing nested XML documents. Subclasses implement `Start()`, `HandleAttribute()`, `HandleString()`, and `End()` to process specific element types. `ResetValues()` allows rolling back partial parses on error.

## External Dependencies
- `<vector>` (STL container for child elements)
- `<stdio.h>` (`sscanf` for numerical parsing)
- `cstypes.h` (platform-specific integer types: `int16`, `uint16`, `int32`, `uint32`)
- `XML_GetBooleanValue()` (defined elsewhere; parses boolean strings)
- `StringsEqual()` (defined elsewhere; case-insensitive comparison)
- `DeUTF8*()` functions (defined elsewhere; UTF-8 conversion utilities)
