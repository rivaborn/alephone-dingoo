# Expat/xmlparse/xmlparse.h

## File Purpose
Public API header for the Expat XML parser library. Defines the interface for creating parsers, registering event callbacks, parsing XML input, and handling errors. This is a callback-driven XML parser suitable for streaming or incremental parsing.

## Core Responsibilities
- Parser lifecycle management (creation, configuration, cleanup)
- Event callback registration for XML constructs (elements, text, comments, processing instructions, CDATA, entities, namespaces)
- Streaming/incremental XML parsing with configurable buffer sizes
- External entity and custom encoding support
- Error reporting with parse location tracking
- User data association with parser instances

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_Parser` | typedef | Opaque handle to parser instance |
| `XML_Char` | typedef | Character type (configurable: `wchar_t`, `unsigned short`, or `char` depending on encoding mode) |
| `XML_LChar` | typedef | Character type for literal/debug strings (typically `char`) |
| `XML_Encoding` | struct | Describes custom encoding: byteΓåÆcodepoint mapping, conversion function, release callback |
| `XML_Error` | enum | Parser error codes (23 distinct error conditions) |

## Global / File-Static State
None.

## Key Functions / Methods

### XML_ParserCreate
- **Signature:** `XML_Parser XMLPARSEAPI XML_ParserCreate(const XML_Char *encoding)`
- **Purpose:** Allocates and initializes a new parser instance
- **Inputs:** `encoding` ΓÇô encoding name (e.g., "UTF-8") or `NULL` for auto-detect
- **Outputs/Return:** New `XML_Parser` handle
- **Side effects:** Allocates heap memory; initializes internal state
- **Calls:** (implementation details hidden)
- **Notes:** Must be paired with `XML_ParserFree`; returns `NULL` on memory allocation failure

### XML_ParserCreateNS
- **Signature:** `XML_Parser XMLPARSEAPI XML_ParserCreateNS(const XML_Char *encoding, XML_Char namespaceSeparator)`
- **Purpose:** Creates a parser with namespace processing enabled
- **Inputs:** `encoding` ΓÇô encoding name or `NULL`; `namespaceSeparator` ΓÇô character to join namespace URI with local name (often `':'` or `'\0'`)
- **Outputs/Return:** New `XML_Parser` handle
- **Side effects:** Allocates memory; enables namespace expansion in element/attribute names
- **Notes:** Expands qualified names to `namespace_uri<sep>local_name` format

### XML_Parse
- **Signature:** `int XMLPARSEAPI XML_Parse(XML_Parser parser, const char *s, int len, int isFinal)`
- **Purpose:** Parses a chunk of XML input; main parsing entry point
- **Inputs:** `parser` ΓÇô parser instance; `s` ΓÇô input buffer; `len` ΓÇô number of bytes; `isFinal` ΓÇô 1 if this is the final chunk
- **Outputs/Return:** 1 (success), 0 (fatal error; call `XML_GetErrorCode` for details)
- **Side effects:** Invokes registered callbacks for each XML event encountered; maintains internal state (line/column tracking, entity expansion)
- **Calls:** Registered handlers (`XML_StartElementHandler`, `XML_CharacterDataHandler`, etc.) indirectly
- **Notes:** Can be called multiple times for incremental parsing; last call must have `isFinal=1`; `len` may be 0

### XML_SetElementHandler
- **Signature:** `void XMLPARSEAPI XML_SetElementHandler(XML_Parser parser, XML_StartElementHandler start, XML_EndElementHandler end)`
- **Purpose:** Registers callbacks for element start/end events
- **Inputs:** `start` ΓÇô handler called with element name and attributes; `end` ΓÇô handler called with element name
- **Outputs/Return:** (void)
- **Side effects:** Replaces previously registered handlers

### XML_SetCharacterDataHandler
- **Signature:** `void XMLPARSEAPI XML_SetCharacterDataHandler(XML_Parser parser, XML_CharacterDataHandler handler)`
- **Purpose:** Registers callback for character data (text between tags)
- **Inputs:** `handler` ΓÇô callback receiving buffer pointer and length (not null-terminated)
- **Side effects:** Replaces previous handler
- **Notes:** May be called multiple times for a single text node; buffer not null-terminated

### XML_ParserFree
- **Signature:** `void XMLPARSEAPI XML_ParserFree(XML_Parser parser)`
- **Purpose:** Deallocates parser and all associated resources
- **Inputs:** `parser` ΓÇô handle returned by `XML_ParserCreate` or `XML_ParserCreateNS`
- **Outputs/Return:** (void)
- **Side effects:** Frees all heap memory; parser handle becomes invalid

### XML_GetErrorCode
- **Signature:** `enum XML_Error XMLPARSEAPI XML_GetErrorCode(XML_Parser parser)`
- **Purpose:** Retrieves error code after parse failure
- **Outputs/Return:** Member of `XML_Error` enum (or `XML_ERROR_NONE` if no error)
- **Notes:** Only meaningful after `XML_Parse` or `XML_ParseBuffer` returns 0

**Notes on other functions:**
- Handler registration functions: `XML_SetCommentHandler`, `XML_SetProcessingInstructionHandler`, `XML_SetDefaultHandler`, `XML_SetNamespaceDeclHandler`, etc. all follow same pattern (register callback; no return)
- Location tracking: `XML_GetCurrentLineNumber`, `XML_GetCurrentColumnNumber`, `XML_GetCurrentByteIndex` ΓÇô queryable during or after parse
- External entities: `XML_SetExternalEntityRefHandler`, `XML_ExternalEntityParserCreate` ΓÇô allow nested parsing of referenced documents
- User data: `XML_SetUserData`, `XML_GetUserData` ΓÇô opaque pointer passed to all callbacks
- Buffer mode: `XML_GetBuffer`, `XML_ParseBuffer` ΓÇô alternative to `XML_Parse` for zero-copy scenarios

## Control Flow Notes
**Initialization ΓåÆ Configuration ΓåÆ Parsing ΓåÆ Callbacks ΓåÆ Cleanup**

1. **Init:** `XML_ParserCreate[NS]()` allocates parser
2. **Config:** Zero or more `XML_Set*Handler()` calls register event callbacks; `XML_SetUserData()` optional
3. **Parsing:** Loop calling `XML_Parse()` with successive input chunks
4. **Callbacks:** Parser invokes registered handlers for each XML construct (start element, text, end element, comment, etc.)
5. **Error handling:** On `XML_Parse` failure, call `XML_GetErrorCode()` and `XML_GetCurrentLine/Column/ByteIndex()` for diagnostics
6. **Cleanup:** `XML_ParserFree()` releases resources

Event-driven architecture: parser does not return document tree; instead, application code receives callbacks.

## External Dependencies
- `#include <stddef.h>` ΓÇô for `NULL` and `wchar_t` definition (only when `XML_UNICODE_WCHAR_T` defined)
- No external library dependencies; self-contained C API
