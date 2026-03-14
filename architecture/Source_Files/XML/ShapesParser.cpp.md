# Source_Files/XML/ShapesParser.cpp

## File Purpose
Parses XML `<shape>` elements to extract and validate shape descriptor attributes (collection, CLUT, sequence/frame indices), then composes them into a `shape_descriptor` value. Provides a reusable static parser that multiple callers can configure via pointer/flag injection.

## Core Responsibilities
- Parse XML shape element attributes: `coll`, `clut`, `seq`, and `frame`
- Validate attribute values against engine limits (collection, CLUT, and sequence bounds)
- Compose validated attributes into a shape descriptor using `BUILD_DESCRIPTOR()` and `BUILD_COLLECTION()`
- Support flexible validation: allow `NONE` (uninitialized) values conditionally via `NONE_Is_OK`
- Provide static parser instance and public interface functions for multi-call reuse
- Report parsing errors via `UnrecognizedTag()` and `AttribsMissing()` callbacks

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_ShapesParser | class | Inherits from `XML_ElementParser`; encapsulates shapes parsing state and callbacks |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| ShapesParser | XML_ShapesParser | static | Singleton parser instance reused across multiple callers |

## Key Functions / Methods

### XML_ShapesParser::Start()
- Signature: `bool Start()`
- Purpose: Reset parser state when a new XML shape element begins parsing
- Inputs: None (uses member state)
- Outputs/Return: Always returns `true`
- Side effects: Clears `CollPresent` and `SeqPresent` flags; resets `CLUT` to 0
- Calls: None
- Notes: Called before `HandleAttribute()` is invoked for the element's attributes

### XML_ShapesParser::HandleAttribute(const char *Tag, const char *Value)
- Signature: `bool HandleAttribute(const char *Tag, const char *Value)`
- Purpose: Parse and validate a single XML attribute (`coll`, `clut`, `seq`, or `frame`)
- Inputs: `Tag` = attribute name; `Value` = attribute value string
- Outputs/Return: `true` if attribute recognized and valid; `false` otherwise
- Side effects: On success, updates `Coll`, `CLUT`, or `Seq` members and sets presence flags; calls `UnrecognizedTag()` if tag unrecognized
- Calls: `StringsEqual()`, `ReadBoundedUInt16Value()`, `UnrecognizedTag()`
- Notes: Both `seq` and `frame` write to the same `Seq` member (legacy alias support); bounds checked against `MAXIMUM_COLLECTIONS`, `MAXIMUM_CLUTS_PER_COLLECTION`, and `MAXIMUM_SHAPES_PER_COLLECTION`

### XML_ShapesParser::AttributesDone()
- Signature: `bool AttributesDone()`
- Purpose: Finalize parsing by validating required attributes and composing the shape descriptor
- Inputs: Uses member state (`CollPresent`, `SeqPresent`, `Coll`, `CLUT`, `Seq`, `NONE_Is_OK`); expects `DescPtr` to be set by caller
- Outputs/Return: `true` if valid and descriptor written; `false` if validation fails
- Side effects: Writes shape descriptor to `*DescPtr` via `BUILD_DESCRIPTOR()` / `BUILD_COLLECTION()`; calls `AttribsMissing()` on validation error
- Calls: `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()`, `AttribsMissing()`
- Notes: Requires both `coll` and `seq`/`frame` present, unless `NONE_Is_OK` is true (in which case both absent ΓåÆ `UNONE`); partial presence (only one of `coll` or `seq`) is an error

### Shape_GetParser()
- Signature: `XML_ElementParser *Shape_GetParser()`
- Purpose: Return the static shapes parser instance for XML parsing use
- Inputs: None
- Outputs/Return: Pointer to `ShapesParser` (singleton)
- Side effects: None
- Calls: None
- Notes: Allows multiple callers to request the same parser; callers must use `Shape_SetPointer()` to configure it before parsing

### Shape_SetPointer(shape_descriptor *DescPtr, bool NONE_Is_OK)
- Signature: `void Shape_SetPointer(shape_descriptor *DescPtr, bool NONE_Is_OK = true)`
- Purpose: Configure the parser's target pointer and validation mode before parsing begins
- Inputs: `DescPtr` = address to write the parsed descriptor; `NONE_Is_OK` = whether to accept uninitialized (NONE) values
- Outputs/Return: None
- Side effects: Updates `ShapesParser.DescPtr` and `ShapesParser.NONE_Is_OK`
- Calls: None
- Notes: Must be called before XML parsing of a shape element; uses default parameter `true` for `NONE_Is_OK`

## Control Flow Notes
Standard XML element parser lifecycle: `Start()` ΓåÆ `HandleAttribute()` (called N times) ΓåÆ `AttributesDone()` ΓåÆ descriptor written to target pointer. External code obtains the parser via `Shape_GetParser()`, configures it via `Shape_SetPointer()` with a target descriptor and validation rules, then triggers XML parsing (via XML framework not shown here). Reusable across multiple shape elements by re-calling `Shape_SetPointer()` with a new target before each element.

## External Dependencies
- **Includes**: `cseries.h` (utility macros/functions), `ShapesParser.h` (public interface)
- **Base class**: `XML_ElementParser` (defined elsewhere; provides callback framework)
- **Types**: `shape_descriptor`, `uint16` (defined elsewhere)
- **Functions called (defined elsewhere)**: `StringsEqual()`, `ReadBoundedUInt16Value()`, `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()`, `UnrecognizedTag()`, `AttribsMissing()`
- **Constants (defined elsewhere)**: `MAXIMUM_COLLECTIONS`, `MAXIMUM_CLUTS_PER_COLLECTION`, `MAXIMUM_SHAPES_PER_COLLECTION`, `UNONE`
