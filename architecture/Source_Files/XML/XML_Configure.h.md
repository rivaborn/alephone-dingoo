# Source_Files/XML/XML_Configure.h

## File Purpose

Abstract base class that orchestrates XML file parsing for Marathon engine configuration. Integrates the Expat C parser library with a tree of XML_ElementParser objects to load and apply game configuration from XML files. Subclasses implement data source (GetData) and error handling.

## Core Responsibilities

- Manage Expat XML parser lifecycle and state (create, feed data, destroy)
- Adapt C-style Expat callbacks to C++ instance methods
- Delegate element-specific parsing to XML_ElementParser element tree
- Track and report three categories of errors: read errors, XML parse errors, and interpretation errors
- Maintain a stack of the current active element parser during document traversal
- Coordinate parsing across multiple data chunks (for streaming/large file support)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `XML_Configure` | class | Abstract base class for XML configuration parsers; subclasses provide data source |
| `XML_ElementParser` | class (defined elsewhere) | Base for element-specific parsers; handles individual XML element types |
| `XML_Parser` | typedef (from expat.h) | Opaque parser handle; manages Expat parsing state |

## Global / File-Static State

None (class instance members hold all state).

## Key Functions / Methods

### DoParse
- **Signature:** `bool DoParse()`
- **Purpose:** Main entry point; orchestrates the complete XML parsing workflow
- **Inputs:** None (data supplied via protected Buffer/BufLen/LastOne, populated by GetData)
- **Outputs/Return:** `bool` ΓÇô `true` if parsing completed successfully, `false` if fatal error
- **Side effects:** Calls GetData repeatedly; modifies Parser state; calls element handlers; may trigger error callbacks
- **Calls:** GetData, XML_Parse (Expat), StartElement, EndElement, CharacterData, ReportParseError, RequestAbort
- **Notes:** Pure parsing orchestration; actual element handling delegated to XML_ElementParser tree

### GetData (pure virtual)
- **Signature:** `virtual bool GetData() = 0`
- **Purpose:** Subclass hook to supply the next chunk of XML data
- **Inputs:** None (subclass reads from file, network, buffer, etc.)
- **Outputs/Return:** `bool` ΓÇô `true` if data was read successfully, `false` on read failure
- **Side effects:** Subclass must populate protected members: Buffer (pointer), BufLen (count), LastOne (bool)
- **Notes:** May be called multiple times for streaming parse; on last call, LastOne must be true

### StartElement, EndElement, CharacterData (instance callback methods)
- **Purpose:** Delegate element parsing to current XML_ElementParser; manage element tree traversal
- **Inputs:** XML data (name, attributes, text content as appropriate)
- **Outputs/Return:** void
- **Side effects:** Modify CurrentElement; call XML_ElementParser::Start/End/HandleString/HandleAttribute
- **Notes:** Private; only called via static callback trampolines from Expat

### StaticStartElement, StaticEndElement, StaticCharacterData (static callbacks)
- **Purpose:** C-style callback adapters; convert Expat opaque void* userData to XML_Configure* and dispatch
- **Inputs:** void* userData (cast to this), XML data
- **Outputs/Return:** void
- **Side effects:** Invoke instance methods on the recovered object pointer
- **Notes:** Required by Expat; UserData parameter must be set to this object pointer via XML_SetUserData

### ComposeInterpretError
- **Signature:** `void ComposeInterpretError(const char *Format, ...)`
- **Purpose:** Report a semantic/interpretation error during parsing (e.g., invalid game data value)
- **Inputs:** printf-style format string and variadic arguments
- **Outputs/Return:** void
- **Side effects:** Increments NumInterpretErrors; calls ReportInterpretError; may request abort if error limit reached
- **Notes:** Interpretation errors are distinct from XML syntax errors (handled by ReportParseError)

### GetNumInterpretErrors
- **Signature:** `int GetNumInterpretErrors()`
- **Purpose:** Query count of interpretation errors encountered so far
- **Inputs:** None
- **Outputs/Return:** `int` ΓÇô current error count
- **Side effects:** None
- **Notes:** Allows caller to decide if parsing should be trusted despite error reporting

---

## Control Flow Notes

This class is designed for the **init/config phase** of the Marathon engine:

1. **Init:** Subclass instantiates XML_Configure and sets CurrentElement to a root element parser
2. **Parsing loop:** DoParse() repeatedly calls GetData(), feeds chunks to Expat, and dispatches callbacks
3. **Element traversal:** As Expat encounters opening tags, StartElement navigates the CurrentElement tree (push); on closing tags, EndElement pops (return to parent)
4. **Data collection:** CharacterData feeds text content to the current element parser; attributes are fed via HandleAttribute
5. **Error handling:** If ReportParseError or ComposeInterpretError is called and error count is high, RequestAbort() may suspend or abort parsing
6. **Shutdown:** After DoParse returns, subclass can check GetNumInterpretErrors and decide whether to apply configuration

## External Dependencies

- **expat.h** ΓÇô Expat XML parser library: provides XML_Parser type, XML_Parse, XML_SetUserData, callback typedefs (XML_StartElementHandler, XML_EndElementHandler, XML_CharacterDataHandler)
- **XML_ElementParser.h** ΓÇô Element parser base class (CurrentElement points to instances of this type; method calls: Start, End, HandleAttribute, HandleString, FindChild)
- **Standard C++** ΓÇô constructor, destructor, virtual methods, includes for stdlib functionality (handled by other headers)
