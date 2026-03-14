# Subsystem Overview

## Purpose
Expat is a callback-driven XML parser library that tokenizes and validates XML documents with support for multiple character encodings. It provides incremental/streaming parsing suitable for resource-constrained environments and external entity resolution via child parsers.

## Key Files
| File | Role |
|------|------|
| Expat/xmlparse/xmlparse.cpp | Core parser state machine, callback dispatch, DTD/entity/namespace management |
| Expat/xmlparse/xmlparse.h | Public parser API (creation, handlers, parsing, error reporting) |
| Expat/xmltok/xmltok.cpp | Encoding detection, UTF-8/UTF-16 conversion, tokenizer instantiation |
| Expat/xmltok/xmltok.h | Public tokenizer API, token/state constants, ENCODING struct |
| Expat/xmltok/xmltok_impl.cpp | Core XML tokenization (tags, attributes, references, CDATA, comments) |
| Expat/xmltok/xmltok_impl.h | Byte-type classification enum (BT_*) for character analysis |
| Expat/xmltok/xmlrole.cpp | Prolog state machine; classifies tokens into semantic roles (ENTITY, NOTATION, ATTLIST, etc.) |
| Expat/xmltok/xmlrole.h | XML role enumerations and prolog state structures |
| Expat/xmlparse/hashtable.cpp | Hash table with linear probing for entity/namespace resolution |
| Expat/xmlparse/hashtable.h | Hash table API and data structures |
| Expat/xmltok/asciitab.h | ASCII byte classification lookup table (0x00ΓÇô0x7F) |
| Expat/xmltok/utf8tab.h | UTF-8 byte classification (lead/continuation/invalid) |
| Expat/xmltok/latin1tab.h | Latin-1 byte classification (0x80ΓÇô0xFF) |
| Expat/xmltok/nametab.h | Unicode name-character bitmap and page indices |
| Expat/xmltok/xmldef.h | Platform abstraction for memory allocation (Windows/NSPR/stdlib) |
| Expat/gennmtab/gennmtab.cpp | Build-time utility generating optimized Unicode name-character tables |
| Expat/xmlwf/xmlwf.cpp | XML well-formedness checker; main CLI entry point |
| Expat/xmlwf/xmlfile.cpp | Memory-mapped and stream-based XML file processing |
| Expat/xmlwf/filemap.h | File I/O abstraction interface |
| Expat/xmlwf/win32filemap.cpp | Windows memory-mapped file operations |
| Expat/xmlwf/unixfilemap.cpp | Unix mmap-based file operations |

## Core Responsibilities
- Detect and manage character encodings (UTF-8, UTF-16 LE/BE, ASCII, Latin-1, Windows codepages)
- Tokenize XML markup (tags, attributes, character data, CDATA, comments, PIs, references, declarations)
- Validate XML grammar via prolog state machine (DOCTYPE, ENTITY, NOTATION, ATTLIST, ELEMENT declarations)
- Maintain parser state across prolog, content, CDATA, and epilog sections
- Manage hash tables for fast entity and namespace lookup
- Register and dispatch user-defined callback handlers for parse events (element start/end, character data, comments, PIs, CDATA, namespaces)
- Track parse location (line/column) for error reporting
- Perform O(1) character classification via pre-computed lookup tables
- Support external entity resolution by creating child parsers

## Key Interfaces & Data Flow
**Public Parser API (xmlparse.h):**
- `XML_ParserCreate()` / `XML_ParserCreateNS()` ΓÇô Initialize parser
- `XML_SetElementHandler()`, `XML_SetCharacterDataHandler()`, `XML_SetCommentHandler()`, `XML_SetProcessingInstructionHandler()`, `XML_SetCDATASectionHandler()`, `XML_SetExternalEntityRefHandler()` ΓÇô Register callbacks
- `XML_Parse()` / `XML_ParseBuffer()` ΓÇô Consume input and dispatch events
- `XML_GetErrorCode()`, `XML_ErrorString()`, `XML_GetCurrentLineNumber()` ΓÇô Error and location tracking

**Data Flow:** Raw bytes ΓåÆ Tokenizer (xmltok) ΓåÆ Role classifier (xmlrole) ΓåÆ Parser state machine (xmlparse) ΓåÆ User callbacks

**Consumers:** xmlwf utility, alephone-dingoo game code (for loading XML resources/config), any code calling Expat's public API

## Runtime Role
Not a per-frame game system. Called during initialization/resource loading phases to parse XML configuration and data files. Stateless between parse operations (create parser, parse, destroy).

## Notable Implementation Details
- **Table-driven character classification:** Uses pre-computed lookup tables (asciitab, utf8tab, latin1tab, nametab) for O(1) single-byte lookups during tokenization, avoiding branch overhead on constrained hardware
- **Parameterized tokenizer:** Macro-driven code generation instantiates encoding-specific tokenizer variants at compile time (UTF-8, UTF-16, ASCII) to avoid runtime dispatch overhead
- **Build-time table generation:** gennmtab.cpp generates optimized bitmaps and page indices for Unicode name-character validation; output included as nametab.h
- **Streaming/incremental design:** Supports XML_ParseBuffer() for chunked input processing, reducing memory footprint on resource-constrained platforms
- **Callback-only model:** No intermediate tree building; events delivered directly to registered handlers for minimal memory allocation
- **Child parser support:** External entity references spawn dedicated parser instances with separate state, enabling modular entity resolution
