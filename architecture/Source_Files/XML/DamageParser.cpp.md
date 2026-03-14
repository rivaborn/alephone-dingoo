# Source_Files/XML/DamageParser.cpp

## File Purpose
Implements an XML parser for damage definition elements in the Aleph One game engine. It parses damage configuration attributes (type, flags, base, random, scale) from XML and populates a damage_definition structure. Part of a larger XML configuration system.

## Core Responsibilities
- Parse and validate individual XML damage element attributes
- Convert attribute values to appropriate types (int16, float, fixed-point)
- Enforce value constraints via bounded validation
- Provide a singleton parser instance accessible to the XML parsing system
- Support multiple damage definitions by allowing pointer reassignment

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_DamageParser | class | Inherits from XML_ElementParser; parses damage element attributes and stores target damage_definition pointer |
| damage_definition | struct (defined elsewhere) | Target structure holding parsed damage data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| DamageParser | XML_DamageParser | static | Singleton parser instance; reused across all damage parsing calls |

## Key Functions / Methods

### XML_DamageParser::HandleAttribute
- **Signature:** `bool HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Parse a single XML attribute and store its value in the target damage_definition structure.
- **Inputs:** `Tag` (attribute name as string), `Value` (attribute value as string)
- **Outputs/Return:** `true` if attribute was recognized and validated; `false` otherwise
- **Side effects:** Modifies fields in `DamagePtr->` (type, flags, base, random, scale) based on attribute tag
- **Calls:** `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`, `ReadBoundedNumericalValue()`, `UnrecognizedTag()`
- **Notes:** 
  - Handles 5 attributes: type, flags, base, random, scale
  - Type is constrained to [NONE, NUMBER_OF_DAMAGE_TYPESΓÇô1]
  - Flags constrained to [0, 1]
  - Scale is converted from float to fixed-point using `FIXED_ONE` with rounding
  - Returns false on unrecognized tag after calling `UnrecognizedTag()`

### Damage_GetParser
- **Signature:** `XML_ElementParser *Damage_GetParser()`
- **Purpose:** Return the singleton damage parser to the XML parsing framework.
- **Inputs:** None
- **Outputs/Return:** Pointer to static DamageParser instance
- **Side effects:** None
- **Calls:** None
- **Notes:** Called multiple times during config loading to reuse the same parser for different damage elements

### Damage_SetPointer
- **Signature:** `void Damage_SetPointer(damage_definition *DamagePtr)`
- **Purpose:** Set the target damage structure that the parser will populate.
- **Inputs:** Pointer to a damage_definition to receive parsed values
- **Outputs/Return:** None
- **Side effects:** Modifies DamageParser.DamagePtr (static field)
- **Calls:** None
- **Notes:** Must be called before XML parsing begins for each damage element

## Control Flow Notes
This file operates in the **initialization/config-loading phase**. Typical flow:
1. XML parser encounters a `<damage>` element
2. Engine calls `Damage_SetPointer()` to target a damage_definition struct
3. Engine retrieves parser via `Damage_GetParser()` and invokes its `HandleAttribute()` for each XML attribute
4. Parser validates and converts values into the target structure
5. Cycle repeats for next damage element

## External Dependencies
- **Framework:** `XML_ElementParser` (base class, defined elsewhere)
- **Utilities:** `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`, `ReadBoundedNumericalValue()`, `UnrecognizedTag()` (defined elsewhere, likely in XML parsing framework)
- **Constants:** `NONE`, `NUMBER_OF_DAMAGE_TYPES`, `FIXED_ONE`, `SHRT_MIN`, `SHRT_MAX` (defined elsewhere)
- **Includes:** `cseries.h` (umbrella header), `DamageParser.h` (public interface), standard C libs (`string.h`, `limits.h`)
