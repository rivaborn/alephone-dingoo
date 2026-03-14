# Expat/xmlparse/xmlparse.cpp

## File Purpose
Core implementation of the Expat XML parser. Manages parser creation/destruction, callback registration, state machines for parsing different XML sections (prolog, content, CDATA, epilog), and memory management for buffers, strings, and DTD information. Handles encoding detection, namespace processing, and external entity references.

## Core Responsibilities
- Create and destroy parser instances with optional namespace support
- Register and invoke callback handlers for XML events (start/end elements, character data, comments, PIs, etc.)
- Manage input buffer and incremental parsing state
- Maintain string memory pools and allocate tag/binding/entity structures
- Store and lookup DTD declarations (elements, attributes, entities, notations)
- Track namespace declarations and bindings with prefix management
- Convert between different character encodings (UTF-8, UTF-16, custom)
- Process external general entities by creating child parsers
- Maintain a stack of open tags and namespace scopes

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Parser | struct | Main parser state: buffers, handlers, encoding, DTD, tag/binding stacks, pools |
| BINDING | struct | Namespace URI binding for a prefix; linked list per tag level |
| PREFIX | struct | Namespace prefix name and current binding |
| TAG_NAME | struct | Element name with namespace-split components (str, localPart, uriLen) |
| TAG | struct | Stack frame for open element: parent, raw name, parsed name, bindings |
| ENTITY | struct | Entity declaration: name, text/system ID, notation, open flag |
| STRING_POOL | struct | Memory pool of BLOCK chains; efficient string allocation |
| BLOCK | struct | Allocator block with size and flexible array member |
| DTD | struct | Parsed DTD info: entity/element/attribute/prefix tables, pool, base URI |
| ATTRIBUTE_ID | struct | Attribute name lookup entry with prefix, xmlns flag, tokenization hint |
| DEFAULT_ATTRIBUTE | struct | Default value for an attribute of an element type |
| ELEMENT_TYPE | struct | Element declaration from DTD: name, prefix, default attributes |
| OPEN_INTERNAL_ENTITY | struct | Stack of open internal entity parsing contexts |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| prologProcessor | static Processor* | file-static function pointer | Initial prolog parser state machine |
| prologInitProcessor | static Processor* | file-static function pointer | Prolog init phase state machine |
| contentProcessor | static Processor* | file-static function pointer | Main element content parser state machine |
| cdataSectionProcessor | static Processor* | file-static function pointer | CDATA section parser state machine |
| epilogProcessor | static Processor* | file-static function pointer | Epilog (post-root) parser state machine |
| errorProcessor | static Processor* | file-static function pointer | Error recovery state machine |
| externalEntityInitProcessor* | static Processor* | file-static function pointers | External entity parsing state machines |

## Key Functions / Methods

### XML_ParserCreate
- Signature: `XML_Parser XML_ParserCreate(const XML_Char *encodingName)`
- Purpose: Allocate and initialize a new parser instance
- Inputs: Optional encoding name (e.g., "UTF-8"); NULL uses auto-detection
- Outputs/Return: New XML_Parser handle or NULL on allocation failure
- Side effects: Allocates Parser struct, internal buffers (data buffer, attribute array), string pools, DTD; registers initial state machine
- Calls: malloc, XmlInitEncoding, poolInit, dtdInit
- Notes: No handlers set by default; caller must register handlers before parsing. Prolog state starts as prologInitProcessor.

### XML_ParserCreateNS
- Signature: `XML_Parser XML_ParserCreateNS(const XML_Char *encodingName, XML_Char nsSep)`
- Purpose: Create parser with namespace processing enabled
- Inputs: Encoding name; namespace separator character (e.g., '|', NUL for concatenation)
- Outputs/Return: New parser or NULL
- Side effects: Calls XML_ParserCreate, then enables namespace mode and sets separator; initializes implicit XML namespace context
- Calls: XML_ParserCreate, setContext, XmlInitEncodingNS
- Notes: Uses namespace-aware internal encoding; sets an implicit xmlns="http://www.w3.org/XML/1998/namespace" binding

### XML_ExternalEntityParserCreate
- Signature: `XML_Parser XML_ExternalEntityParserCreate(XML_Parser oldParser, const XML_Char *context, const XML_Char *encodingName)`
- Purpose: Create a child parser for an external general entity
- Inputs: Parent parser; context string (namespace/entity bindings); encoding for entity
- Outputs/Return: New parser with copied handlers/DTD or NULL
- Side effects: Creates new parser, copies handlers/userData/DTD from parent, sets initial processor to externalEntityInitProcessor
- Calls: XML_ParserCreate[NS], dtdCopy, setContext
- Notes: New parser is independent; can be used in separate thread. Context encodes active namespaces and open entities as a NUL-terminated string with formfeed separators.

### XML_ParserFree
- Signature: `void XML_ParserFree(XML_Parser parser)`
- Purpose: Deallocate all parser resources
- Inputs: Parser handle (may be NULL)
- Outputs/Return: None
- Side effects: Frees all tags, bindings, DTD, pools, buffers, encoding data; safe for repeated calls on same parser
- Calls: destroyBindings, poolDestroy, dtdDestroy, free
- Notes: Handles both active tag stack and free list to prevent leaks

### XML_SetElementHandler
- Signature: `void XML_SetElementHandler(XML_Parser parser, XML_StartElementHandler start, XML_EndElementHandler end)`
- Purpose: Register callbacks for element start/end events
- Inputs: Parser; start callback (NULL to unregister); end callback
- Outputs/Return: None
- Side effects: Updates parser's startElementHandler and endElementHandler fields
- Calls: (none; direct assignment via macros)
- Notes: Start handler receives element name and attribute array; end handler receives element name only

### poolGrow
- Signature: `static int poolGrow(STRING_POOL *pool)`
- Purpose: Allocate or expand the memory pool when full
- Inputs: Pool pointer
- Outputs/Return: 1 on success, 0 on allocation failure
- Side effects: Allocates new BLOCK, memcpy's existing data, updates pool pointers; tries to reuse freeBlocks first
- Calls: malloc, realloc, memcpy
- Notes: Doubles block size on exponential growth; offsets BLOCK header overhead when allocating

### dtdCopy
- Signature: `static int dtdCopy(DTD *newDtd, const DTD *oldDtd)`
- Purpose: Deep copy all DTD tables (entities, elements, attributes, prefixes)
- Inputs: Destination DTD (pre-initialized); source DTD
- Outputs/Return: 1 on success, 0 on allocation failure
- Side effects: Populates newDtd's hash tables and pool with copies of oldDtd entries
- Calls: poolCopyString, poolCopyStringN, lookup (for each table)
- Notes: Used when creating external entity parser to inherit parent's DTD context. Maintains cross-references (e.g., attribute prefix ΓåÆ copied prefix struct).

### setContext
- Signature: `static int setContext(XML_Parser parser, const XML_Char *context)`
- Purpose: Parse and apply a context string (namespace bindings and open entities)
- Inputs: Parser; context string (tokens separated by formfeed; "prefix=uri" or "=uri" for default, or entity name)
- Outputs/Return: 1 on success, 0 on allocation failure
- Side effects: Calls addBinding for each namespace; marks entities as open
- Calls: lookup, addBinding, poolAppendChar
- Notes: Used by setContext to establish namespace scope for external entity parsing

### normalizeLines
- Signature: `static void normalizeLines(XML_Char *s)`
- Purpose: Normalize line endings (CR, CRLF ΓåÆ LF)
- Inputs: In-place string pointer
- Outputs/Return: None
- Side effects: Modifies string in-place; shortens it if CR sequences present
- Calls: (none)
- Notes: Applied to comment and PI data before passing to handlers

### reportProcessingInstruction
- Signature: `static int reportProcessingInstruction(XML_Parser parser, const ENCODING *enc, const char *start, const char *end)`
- Purpose: Extract PI target and data, then invoke callback or default handler
- Inputs: Parser; encoding; raw PI bytes (inclusive of <? and ?>)
- Outputs/Return: 1 on success, 0 on allocation failure
- Side effects: Parses and stores PI data in tempPool; calls processingInstructionHandler or defaultHandler; clears tempPool
- Calls: poolStoreString, poolFinish, poolClear, normalizeLines
- Notes: Skips PI markers and whitespace; normalizes line endings in data

## Control Flow Notes
The parser operates as a state machine via function pointers:
- **Prolog phase** (prologInitProcessor ΓåÆ prologProcessor): Scans XML declaration, DOCTYPE, process before root element
- **Content phase** (contentProcessor): Parses element start tags, character data, end tags, CDATA, comments, PIs
- **CDATA phase** (cdataSectionProcessor): Consumes character data until ]]>
- **Epilog phase** (epilogProcessor): Accepts comments/PIs after document element
- **External entity** (externalEntityInitProcessor*): Similar phases for nested entity parsing
- **Error phase** (errorProcessor): Stops parsing on fatal error

Each processor is invoked by the public parse entry points (not shown in excerpt) with slices of input buffer.

## External Dependencies
- `xmldef.h`: Memory allocation macros (malloc, free, realloc), platform abstractions
- `xmlparse.h`: Public API types (XML_Parser, handler typedefs, error codes)
- `xmltok.h`: Encoding structures (ENCODING, INIT_ENCODING), token/state constants, encoding functions (XmlConvert, XmlEncode, etc.)
- `xmlrole.h`: XML grammar role/state machine (PROLOG_STATE, XmlPrologStateInit)
- `hashtable.h`: Hash table API (HASH_TABLE, lookup, hashTableInit, hashTableIterNext)
- Standard C: string.h, stdlib.h (via xmldef.h)
