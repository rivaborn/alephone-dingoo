# Expat/xmlwf/xmlfile.cpp

## File Purpose
Processes XML files from disk for the Expat parser's `xmlwf` (well-formedness checker) utility. Provides two parsing strategies: file-mapped (memory-efficient for local files) and stream-based (general-purpose). Manages external entity resolution and delegates to the Expat parser core.

## Core Responsibilities
- Read XML files from disk and feed content to the Expat parser
- Support two processing modes: file-mapped mode and stream-based mode
- Create and manage sub-parsers for external entity references
- Resolve relative system IDs (file paths) for external entities
- Report parsing errors with line and column information
- Handle I/O errors and memory allocation failures

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PROCESS_ARGS | struct | Passes parser and result pointer through the file processing callback chain |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| READ_SIZE | macro | static | Buffer size for stream-based reading (16 bytes debug, 8 KB release) |

## Key Functions / Methods

### reportError
- **Signature:** `static void reportError(XML_Parser parser, const XML_Char *filename)`
- **Purpose:** Display parsing errors with location information
- **Inputs:** parser (Expat parser instance), filename (source file being parsed)
- **Outputs/Return:** None (writes to stdout/stderr)
- **Side effects:** Writes error message to stderr or stdout via `ftprintf()`
- **Calls:** `XML_GetErrorCode()`, `XML_ErrorString()`, `XML_GetErrorLineNumber()`, `XML_GetErrorColumnNumber()`
- **Notes:** Falls back to generic error message if XML_ErrorString returns null

### processFile
- **Signature:** `static void processFile(const void *data, size_t size, const XML_Char *filename, void *args)`
- **Purpose:** Callback invoked by `filemap()` to parse a chunk of file-mapped memory
- **Inputs:** data (buffer), size (bytes), filename, args (PROCESS_ARGS struct)
- **Outputs/Return:** None (modifies result via pointer in args)
- **Side effects:** Updates `*retPtr` in args struct; calls `XML_Parse()`
- **Calls:** `XML_Parse()`, `reportError()`
- **Notes:** Used only in file-mapped mode; `isFinal=1` indicates complete file

### resolveSystemId
- **Signature:** `static const XML_Char *resolveSystemId(const XML_Char *base, const XML_Char *systemId, XML_Char **toFree)`
- **Purpose:** Resolve relative file paths for external entities
- **Inputs:** base (base path from XML_SetBase), systemId (entity reference), toFree (output alloc flag)
- **Outputs/Return:** Resolved path string; pointer to free'd data if allocated
- **Side effects:** Allocates memory if path is relative; caller must free via `*toFree`
- **Calls:** `tcslen()`, `tcscpy()`, `tcsrchr()`, `malloc()`
- **Notes:** Returns systemId unchanged if absolute or no base; Windows: recognizes drive letters (C:) and backslash separators

### externalEntityRefFilemap / externalEntityRefStream
- **Signature:** `static int externalEntityRefFilemap(XML_Parser parser, const XML_Char *context, const XML_Char *base, const XML_Char *systemId, const XML_Char *publicId)`
- **Purpose:** Handler for external entity references; creates sub-parser and processes referenced file
- **Inputs:** parser (parent), context (parse state), base (base URI), systemId (entity path), publicId
- **Outputs/Return:** 1 if success, 0 on error
- **Side effects:** Creates sub-parser, allocates memory, loads and parses external file, frees sub-parser
- **Calls:** `XML_ExternalEntityParserCreate()`, `resolveSystemId()`, `XML_SetBase()`, `filemap()` or `processStream()`, `XML_ParserFree()`, `free()`
- **Notes:** Filemap variant uses memory-mapped I/O; stream variant uses chunked file reads. Both are registered as external entity handlers based on flags.

### processStream
- **Signature:** `static int processStream(const XML_Char *filename, XML_Parser parser)`
- **Purpose:** Read file via buffered I/O and feed to parser in chunks
- **Inputs:** filename, parser
- **Outputs/Return:** 1 if parse succeeded, 0 on any error
- **Side effects:** Opens file, allocates parser buffers, reads from disk, closes file; calls `XML_ParseBuffer()`
- **Calls:** `topen()`, `XML_GetBuffer()`, `read()`, `XML_ParseBuffer()`, `reportError()`, `close()`
- **Notes:** Loop continues until `nread == 0` (EOF). Exits early on parse error or I/O error.

### XML_ProcessFile (public entry point)
- **Signature:** `int XML_ProcessFile(XML_Parser parser, const XML_Char *filename, unsigned flags)`
- **Purpose:** Main dispatcher to process an XML file with the given parser and options
- **Inputs:** parser (Expat instance), filename, flags (`XML_MAP_FILE`, `XML_EXTERNAL_ENTITIES`)
- **Outputs/Return:** 1 if success, 0 on error
- **Side effects:** Calls `XML_SetBase()`, registers external entity handlers, reads and parses file
- **Calls:** `XML_SetBase()`, `XML_SetExternalEntityRefHandler()`, `filemap()`, `processStream()`
- **Notes:** Flags determine parsing mode and whether external entities are processed. Exits with error code 1 if `XML_SetBase()` fails (OOM).

## Control Flow Notes
**Initialization/Entry:** `XML_ProcessFile()` is the public entry point, called once per file to validate.  
**Frame/Update:** Not applicableΓÇöthis is a synchronous file processing utility.  
**Shutdown:** Parsers are freed after processing completes.

External entities trigger sub-parsers via registered handlers (`externalEntityRefFilemap` or `externalEntityRefStream`), creating a recursive parse tree for DTD and entity references.

## External Dependencies
- **Includes/Imports:** `<stdio.h>`, `<stdlib.h>`, `<fcntl.h>`, `<unistd.h>` (POSIX), `<io.h>` (MSVC)
- **Expat API:** `XML_Parser`, `XML_Parse()`, `XML_ParseBuffer()`, `XML_GetBuffer()`, `XML_SetBase()`, `XML_ExternalEntityParserCreate()`, `XML_ParserFree()`, `XML_GetErrorCode()`, `XML_ErrorString()`, `XML_GetErrorLineNumber()`, `XML_GetErrorColumnNumber()`, `XML_SetExternalEntityRefHandler()`
- **Local modules:** `filemap.h` (file-mapping abstraction), `xmltchar.h` (encoding macros), `xmlfile.h` (public interface)
- **System I/O:** `open()`, `read()`, `close()` (POSIX), plus Windows macros via `xmltchar.h` for wide-char support
