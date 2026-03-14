# Expat/xmlwf/xmlwf.cpp

## File Purpose
Main executable for `xmlwf`, an XML well-formedness checker and formatter built on the Expat XML parser. Processes XML files from the command line and outputs them in multiple formats (normalized XML, metadata with parse event details, or timing-only mode), with optional namespace processing and Windows code page support.

## Core Responsibilities
- **Character data escaping**: Escape special XML characters (`&`, `<`, `>`, `"`, whitespace) for safe output
- **Element formatting**: Output opening and closing tags with sorted attributes, in both standard and namespace-aware modes
- **Meta/debug output**: Generate detailed parse event XML (start/end tags, character data, comments, CDATA, entity decls, etc.)
- **Processing instruction handling**: Pass through PI targets and data
- **Encoding support**: Handle unknown encodings (Windows code pages) via callback mapping
- **Command-line interface**: Parse flags for output mode, encoding, namespaces, and input/output file handling
- **Multi-file processing**: Loop over input files, create/configure/destroy parsers, manage output streams

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_Parser | opaque typedef | Expat parser instance (created per file) |
| XML_Encoding | struct | Character mapping and encoding conversion info (from xmlparse.h) |
| FILE | struct | Output stream for formatted XML or metadata |

## Global / File-Static State
None. All state is local to `tmain()` or passed via callback closures (parser's userData pointer carries FILE*).

## Key Functions / Methods

### characterData
- **Signature:** `static void characterData(void *userData, const XML_Char *s, int len)`
- **Purpose:** Write character data to output stream, escaping XML special characters
- **Inputs:** userData (FILE* for output), s (character buffer, not null-terminated), len (byte count)
- **Outputs/Return:** void
- **Side effects:** Writes escaped characters to FILE* via fputts/ftprintf/puttc
- **Calls:** fputts, ftprintf, puttc
- **Notes:** Handles `&`, `<`, `>`, `"` as entity references; whitespace (chars 9, 10, 13) as numeric character references; other chars pass through

### startElement
- **Signature:** `static void startElement(void *userData, const XML_Char *name, const XML_Char **atts)`
- **Purpose:** Output opening tag with sorted attributes in standard (non-namespace) mode
- **Inputs:** userData (FILE*), name (element local name), atts (array of name/value pairs, null-terminated)
- **Outputs/Return:** void
- **Side effects:** Writes `<name attr1="val1" attr2="val2">` to FILE*
- **Calls:** puttc, fputts, qsort (via attcmp comparator), characterData
- **Notes:** Sorts attributes lexicographically; escapes attribute values via characterData

### endElement
- **Signature:** `static void endElement(void *userData, const XML_Char *name)`
- **Purpose:** Output closing tag
- **Inputs:** userData (FILE*), name (element name)
- **Outputs/Return:** void
- **Side effects:** Writes `</name>` to FILE*
- **Calls:** puttc, fputts

### startElementNS / endElementNS
- **Signature:** `static void startElementNS(void *userData, const XML_Char *name, const XML_Char **atts)` / `static void endElementNS(void *userData, const XML_Char *name)`
- **Purpose:** Namespace-aware equivalents; expand qualified names with namespace URIs and generate xmlns declarations
- **Inputs:** name parameter contains "uri#local" (NSSEP='#' as separator); atts same format
- **Outputs/Return:** void
- **Side effects:** Writes opening/closing tags with `ns0:`, `xmlns:ns0="uri"` attributes
- **Calls:** tcsrchr (find NSSEP), fputts, ftprintf, characterData, qsort
- **Notes:** Extracts namespace URI prefix and generates synthetic namespace declarations; increments nsi counter for additional namespaces

### processingInstruction
- **Signature:** `static void processingInstruction(void *userData, const XML_Char *target, const XML_Char *data)`
- **Purpose:** Output processing instruction
- **Inputs:** userData (FILE*), target, data (both null-terminated)
- **Outputs/Return:** void
- **Side effects:** Writes `<?target data?>` to FILE*
- **Calls:** puttc, fputts

### metaStartElement, metaEndElement, metaCharacterData, metaComment, etc.
- **Purpose:** Family of metadata output handlers (enabled with `-m` flag); emit XML elements describing parse events
- **Inputs:** XML_Parser and event-specific data
- **Outputs/Return:** void
- **Side effects:** Write XML metadata tags to FILE* (e.g., `<starttag>`, `<endtag>`, `<chars>`)
- **Calls:** ftprintf, fputts, characterData, metaLocation (appends byte/line/col/uri info)
- **Notes:** metaLocation queries parser state (XML_GetCurrentByteIndex, etc.); handles specified vs. defaulted attributes

### unknownEncoding
- **Signature:** `static int unknownEncoding(void *userData, const XML_Char *name, XML_Encoding *info)`
- **Purpose:** Handler for unknown encodings; maps Windows code page names (e.g., "windows-1252") to character maps
- **Inputs:** userData (ignored), name (encoding name string), info (XML_Encoding struct to populate)
- **Outputs/Return:** 1 if mapped successfully, 0 otherwise
- **Side effects:** Calls malloc to allocate int for code page; fills info->map, info->convert, info->release, info->data
- **Calls:** codepageMap (defined elsewhere), malloc, free (set as release callback)
- **Notes:** Parses "windows-####" format; validates code page number < 0x10000; requires xmltchar.h macros for tcsrchr, tcschr, etc.

### tmain (main entry point)
- **Signature:** `int tmain(int argc, XML_Char **argv)`
- **Purpose:** Parse command-line arguments and process input XML files
- **Inputs:** argc, argv (command-line arguments)
- **Outputs/Return:** 0 on success (or exit(1) on usage error)
- **Side effects:** Creates/destroys parsers, opens/closes input/output files, writes formatted XML or metadata to files
- **Calls:** XML_ParserCreate[NS], XML_SetElementHandler, XML_SetCharacterDataHandler, XML_SetUnknownEncodingHandler, XML_ProcessFile, XML_ParserFree, tfopen, fclose, free, usage
- **Notes:** 
  - Flags: `-n` (namespaces), `-m` (meta), `-c` (compact/default), `-t` (timing only), `-d DIR` (output dir), `-e ENC` (encoding), `-s` (standalone check), `-w` (Windows code pages), `-x` (external entities), `-r` (no file mapping)
  - Each file gets a fresh parser; output mode and namespace handling configured per parse
  - Optional output file written if `-d` specified; deleted on parse failure
  - Handlers configured based on outputType: 'm' ΓåÆ meta handlers, 'c' ΓåÆ default handlers, 't' ΓåÆ no-op handlers, default ΓåÆ normal handlers

## Control Flow Notes
**Initialization ΓåÆ Per-File Loop ΓåÆ Per-Parser Configuration ΓåÆ Parse ΓåÆ Cleanup:**
1. `tmain()` parses flags and validates input files
2. For each input file:
   - Create parser (standard or namespace-aware via XML_ParserCreateNS)
   - Open optional output file (directory + input basename)
   - Install handlers based on output mode (-m/-c/-t or default)
   - Call XML_ProcessFile to read and parse, invoking handlers for each event
   - Close output file and free parser
3. If parse fails (XML_ProcessFile returns 0), delete output file

Handlers run during parse and format/output events directly.

## External Dependencies
- **Standard C:** stdio.h, stdlib.h, string.h, stddef.h (I/O, memory, string functions)
- **Expat parser:** xmlparse.h (XML_Parser, XML_ParserCreate, handlers, query functions)
- **Project-local:** xmlfile.h (XML_ProcessFile), codepage.h (codepageMap, codepageConvert), xmltchar.h (platform/Unicode macros: T, ftprintf, tfopen, tcscmp, tcsrchr, etc.)
- **External symbols referenced:**
  - XML_* (parser API): Create, Free, SetUserData, SetElementHandler, SetCharacterDataHandler, SetProcessingInstructionHandler, SetCommentHandler, SetCdataSectionHandler, SetDefaultHandler, SetUnparsedEntityDeclHandler, SetNotationDeclHandler, SetNamespaceDeclHandler, SetNotStandaloneHandler, SetUnknownEncodingHandler, UseParserAsHandlerArg
  - XML_* (query): GetUserData, GetBase, GetCurrentByteIndex, GetCurrentByteCount, GetCurrentLineNumber, GetCurrentColumnNumber, GetSpecifiedAttributeCount, DefaultCurrent
  - codepageMap, codepageConvert (platform code page support, defined elsewhere)
