# Source_Files/XML/XML_ResourceFork.h

## File Purpose
A C++ class wrapper for parsing XML-encoded data stored in MacOS resource forks. Extends the generic `XML_Configure` base class to handle resource fork I/O, allowing Marathon engine configuration files stored as MacOS resources to be parsed into the game engine's configuration system.

## Core Responsibilities
- Parse single XML resources by type and ID
- Parse all XML resources of a given type from a resource fork
- Manage a resource handle and maintain lifecycle of resource data
- Implement platform-specific I/O for MacOS resource fork access
- Report parsing errors (read errors, XML syntax errors, interpretation errors) with source context
- Provide hooks to abort parsing on excessive errors

## Key Types / Data Structures
None (only inherits from `XML_Configure`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ResourceHandle` | `Handle` | instance (private) | Opaque handle to the in-memory resource data |
| `SourceName` | `char*` | instance (public) | Pointer to a C-string naming the XML source for error reporting |

## Key Functions / Methods

### ParseResource
- Signature: `bool ParseResource(ResType Type, short ID)`
- Purpose: Parse a single resource fork resource containing XML data
- Inputs: `Type` (4-char resource type), `ID` (numeric resource identifier)
- Outputs/Return: `bool` indicating whether the resource exists and was successfully parsed
- Side effects: Modifies internal parser state; calls `GetData()` to load resource bytes; may call error report methods
- Calls: Inherited `DoParse()` from `XML_Configure`
- Notes: Entry point for single-resource parsing; success depends on both resource existence and XML validity

### ParseResourceSet
- Signature: `bool ParseResourceSet(ResType Type)`
- Purpose: Parse all resources of a given type in the resource fork
- Inputs: `Type` (4-char resource type)
- Outputs/Return: `bool` indicating whether any resources of that type were found
- Side effects: Iterates over matching resources; calls `GetData()` for each; updates parser state repeatedly
- Calls: Inherited `DoParse()` from `XML_Configure` (once per resource)
- Notes: Aggregates all resources of one type; reports overall success if at least one was found

### GetData (private)
- Signature: `bool GetData()`
- Purpose: Fetch the next chunk of XML data from the resource for parsing
- Outputs/Return: `bool` indicating read success
- Side effects: Must fill inherited members `Buffer`, `BufLen`, `LastOne`
- Notes: Called repeatedly by `XML_Configure::DoParse()` until `LastOne` is true

### Error Reporting Methods (private)
- `ReportReadError()` ΓÇô Resource fork I/O failure
- `ReportParseError(const char *ErrorString, int LineNumber)` ΓÇô XML syntax error
- `ReportInterpretError(const char *ErrorString)` ΓÇô Semantic/validation error
- `RequestAbort()` ΓÇô Query whether to abort parsing due to accumulated errors

## Control Flow Notes
Initialization ΓåÆ Filetype-based resource lookup ΓåÆ `ParseResource`/`ParseResourceSet` ΓåÆ loop: `GetData()` then `DoParse()` ΓåÆ error handling ΓåÆ cleanup. Fits into a resource-loading phase, likely during engine startup or configuration hot-reload.

## External Dependencies
- **Inheritance:** `XML_Configure` (XML parser base; uses expat via static callbacks, `XML_ElementParser` hierarchy)
- **Platform types:** `Handle`, `ResType`, `short` (MacOS/Carbon resource manager API)
- **Includes:** `XML_Configure.h` only

**Notes:**
- Constructor initializes pointers to NULL (safe default)
- `SourceName` is a convenience field for error messages, aiding debugging of misconfigured resource files
- No explicit memory management visible; assumes resource handle lifecycle is managed by caller
