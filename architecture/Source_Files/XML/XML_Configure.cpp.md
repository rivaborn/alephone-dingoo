# Source_Files/XML/XML_Configure.cpp

## File Purpose
Implementation of XML configuration file parser for the Marathon/Aleph One game engine. Orchestrates parsing via the Expat XML library, manages element state transitions, and collects interpretation errors during config file validation.

## Core Responsibilities
- Provide static callback wrappers that forward parser events to instance methods
- Manage XML element tree traversal (StartElement finds/activates children, EndElement traverses back to parent)
- Call user-defined element handlers (Start, End, HandleAttribute, HandleString) at appropriate parse points
- Perform parsing loop: repeatedly read data chunks via virtual GetData(), feed to Expat, handle parse errors
- Format and report interpretation errors (validation failures on attributes, structure, etc.)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_Configure | class | Main parser orchestrator; subclasses implement GetData() and error reporting hooks |
| XML_ElementParser | class (defined elsewhere) | Represents a single XML element node; maintains Parent pointer and provides element-specific parsing logic |
| XML_Parser | opaque type (Expat) | Underlying C parser from libexpat; created per parsing session |

## Global / File-Static State
None.

## Key Functions / Methods

### StaticStartElement, StaticEndElement, StaticCharacterData
- **Signature:** `static void Static*(void *UserData, const char *Name, ...)`
- **Purpose:** Expat callback wrappers that typecast UserData back to `this` and forward to instance methods
- **Inputs:** UserData (XML_Configure*), Name (tag name), remaining args vary (Attributes array, String/Length)
- **Outputs/Return:** void
- **Side effects:** Call instance methods that mutate CurrentElement and may log errors
- **Calls:** StartElement(), EndElement(), CharacterData()

### StartElement
- **Signature:** `void StartElement(const char *Name, const char **Attributes)`
- **Purpose:** Handle opening tag; find child element in tree, activate it, call its Start() handler, and process all attributes
- **Inputs:** Name (child element tag), Attributes (null-terminated array of keyΓÇôvalue pairs)
- **Outputs/Return:** void
- **Side effects:** Modifies CurrentElement (sets to child, child's Parent set to old current), calls child's Start() and HandleAttribute() methods, logs errors
- **Calls:** CurrentElement->FindChild(), CurrentElement->Start(), CurrentElement->HandleAttribute(), CurrentElement->AttributesDone(), ComposeInterpretError()
- **Notes:** If child not found or handlers fail, error is logged but tree is not traversed (CurrentElement not updated). Iterates Attributes array in pairs (tag, value).

### EndElement
- **Signature:** `void EndElement(const char *Name)`
- **Purpose:** Handle closing tag; finalize current element and traverse back to parent
- **Inputs:** Name (expected closing tag name)
- **Outputs/Return:** void
- **Side effects:** Calls CurrentElement->End(), sets CurrentElement to parent, logs errors
- **Calls:** CurrentElement->NameMatch(), CurrentElement->End(), ComposeInterpretError()
- **Notes:** Early return if Name does not match current element's name (handles unrecognized child case gracefully). Parent pointer is assumed valid.

### CharacterData
- **Signature:** `void CharacterData(const char *String, int Length)`
- **Purpose:** Handle text content within current element
- **Inputs:** String (text fragment, not null-terminated), Length (byte count)
- **Outputs/Return:** void
- **Side effects:** Calls CurrentElement->HandleString(), logs errors
- **Calls:** CurrentElement->HandleString(), ComposeInterpretError()

### DoParse
- **Signature:** `bool DoParse()`
- **Purpose:** Main entry point; orchestrate entire parse cycle from data loading through error reporting
- **Inputs:** None (reads from protected members Buffer, BufLen, LastOne set by GetData())
- **Outputs/Return:** bool (true if parsing and interpretation succeeded, false if any error)
- **Side effects:** Creates/destroys XML_Parser, repeatedly calls virtual GetData(), updates NumInterpretErrors, calls RequestAbort() hook
- **Calls:** XML_ParserCreate(), XML_SetUserData(), XML_SetElementHandler(), XML_SetCharacterDataHandler(), GetData(), XML_Parse(), RequestAbort(), ReportReadError(), ReportParseError(), XML_ParserFree()
- **Notes:** Loop condition `while (!LastOne)` reads chunks until GetData() sets LastOne=true. Expat errors (XML syntax) trigger ReportParseError() and early exit. Interpretation errors (semantic validation) are collected but do not halt parsing immediately. Parser freed on all exit paths.

### ComposeInterpretError
- **Signature:** `void ComposeInterpretError(const char *Format, ...)`
- **Purpose:** Format and report an interpretation error using printf-style format string
- **Inputs:** Format (printf format), ... (variadic args)
- **Outputs/Return:** void
- **Side effects:** Increments NumInterpretErrors, calls virtual ReportInterpretError()
- **Calls:** vsprintf(), ReportInterpretError()
- **Notes:** Uses fixed 256-byte buffer (potential overflow risk if long messages); error count incremented before reporting.

## Control Flow Notes
**Init/Parse Sequence:**
1. Subclass constructs XML_Configure, sets CurrentElement to a root proxy element (contains all valid file-root elements as children)
2. Subclass calls DoParse()
3. DoParse() creates Expat parser, sets callbacks, enters loop
4. Loop calls virtual GetData() (typically reads file chunk, sets Buffer/BufLen/LastOne)
5. Expat parses buffer; fires callbacks for each tag and text
6. Callbacks invoke element handlers and traverse element tree
7. On parse completion or error, parser is freed and result returned

**Error Handling:**
- **Read errors:** Logged via ReportReadError(), parse aborted
- **XML parse errors:** Logged via ReportParseError() with error code and line number, parse aborted
- **Interpretation errors:** Logged via ReportInterpretError(), NumInterpretErrors incremented, parsing continues unless RequestAbort() returns true

## External Dependencies
- **Expat XML library:** `#include "expat.h"` ΓÇö provides XML_Parser, XML_ParserCreate(), XML_Parse(), XML_ErrorString(), XML_GetErrorCode(), XML_GetCurrentLineNumber()
- **XML_ElementParser** (defined elsewhere): `#include "XML_ElementParser.h"` ΓÇö element tree node type with handlers (Start, End, HandleAttribute, HandleString, NameMatch, FindChild, GetName, ErrorString member)
- **Standard library:** stdio.h (unused here), stdarg.h (for va_list/vsprintf in ComposeInterpretError)
- **cseries.h** (project utility headers)
