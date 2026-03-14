# Expat/gennmtab/gennmtab.cpp
## File Purpose
Code generation utility for the Expat XML parser. Generates optimized C bitmap lookup tables for validating XML name characters according to the W3C XML specification. This is a build-time tool, not part of the runtime library.

## Core Responsibilities
- Define Unicode character ranges for valid XML name-start characters (letters, ideographs, etc.)
- Define Unicode character ranges for valid name continuation characters (combining marks, digits, extenders, etc.)
- Populate in-memory lookup tables from character range definitions
- Generate compressed bitmap and page-index tables as C code output
- Optimize table representation by detecting duplicate pages and homogeneous regions

## External Dependencies
- `<string.h>` ΓÇô `memset`, `memcpy`, `memcmp`
- `<stdio.h>` ΓÇô `printf`, `putchar`
- `<stddef.h>` ΓÇô `size_t` type definition

**Defined elsewhere:** XML character classification rules derived from W3C XML 1.0 specification (BaseChar, Ideographic, CombiningChar, Digit, Extender categories).

# Expat/MacOS Support/GetFullPathObject.h
## File Purpose
A utility struct that encapsulates the logic for resolving the full filesystem path of a macOS FSSpec object. Acts as a simple function object wrapper, storing both the resolution result and any errors that occur during the operation.

## Core Responsibilities
- Provides a method to retrieve the full filesystem path from a macOS FSSpec
- Maintains the resolved path as a C string in a custom vector container
- Stores macOS error codes from the path resolution operation
- Initializes error state on construction

## External Dependencies
- `<Carbon.h>` ΓÇô macOS Carbon framework (FSSpec, OSErr)
- `"SimpleVec.h"` ΓÇô Custom dynamic vector template
- `GetFullPath()` implementation defined elsewhere

# Expat/MacOS Support/MiscUtils.h
## File Purpose
A macOS-specific utility header providing inline template functions and helper utilities for mathematical operations, modular arithmetic, and Pascal-to-C string conversions. Intended as a collection of convenience functions for the broader Expat codebase.

## Core Responsibilities
- Provide generic template utilities (squaring function)
- Round floating-point values to integers
- Perform positive-range modular arithmetic with wraparound
- Convert between Pascal and C string formats (legacy macOS string types)
- Document available Standard Template Library functions for reference

## External Dependencies
- `#include <algorithm.h>` ΓÇô Standard Template Library (note: legacy include syntax)
- `#include <MacTypes.h>` ΓÇô macOS/Classic Mac Toolbox types (Str255 definition)
- `using namespace std;` ΓÇô Standard namespace in scope

# Expat/MacOS Support/MoreFilesExtract.h
## File Purpose
A C header file declaring the `FSpGetFullPath` function extracted from the MoreFiles macOS filesystem library. This provides a wrapper for resolving full pathnames from FSSpec records on classic macOS.

## Core Responsibilities
- Declare the `FSpGetFullPath` function interface
- Document parameter semantics and memory ownership
- Enumerate error codes for filesystem resolution failures

## External Dependencies
- **Carbon.h:** Provides `FSSpec`, `OSErr`, `Handle`, and macOS Toolbox filesystem types
- **MoreFiles library:** Original implementation of `FSpGetFullPath` (not defined in this file)

# Expat/MacOS Support/OpenSave_Interface.h
## File Purpose
Defines a cross-platform interface for file open/save dialogs, abstracting platform differences between Win32 and MacOS. Provides parameter structs and two entry-point functions to invoke native dialog boxes and retrieve the user's file selection as a pathname string.

## Core Responsibilities
- Define parameter structures for open and save file dialogs
- Encapsulate platform-specific options (Win32 suffixes, MacOS type/creator codes) in a single interface
- Declare `OpenFile()` and `SaveFile()` functions for invoking platform dialogs
- Document memory management expectations (malloc/free for returned strings)
- Provide guidance on safe file operations (temporary file pattern for saves)

## External Dependencies
- C standard types: `char`, `short`, `unsigned long`.
- C++ extern "C" guards to allow C++ callers to link with C implementations.
- No explicit includes visible.

# Expat/MacOS Support/SimpleVec.h
## File Purpose
Defines custom template-based vector and array containers (`simple_vector` and `simple_array`) as alternatives to STL containers. These containers offer simple resizable storage with STL-like interfaces, designed to work around allocator issues encountered with standard STL containers on some platforms.

## Core Responsibilities
- Provide heap-allocated resizable vector storage via `simple_vector<T>`
- Provide fixed-dimension array-of-vectors storage via `simple_array<T,N>`
- Manage memory allocation and deallocation with explicit `allocate()` / `deallocate()` calls
- Expose STL-compatible interfaces: indexing, iterator-like accessors, assignment, swapping
- Support initialization from raw pointers, copy construction, and range-based construction

## External Dependencies
- `#include <algorithm.h>` ΓÇô provides `copy()` and `swap()` functions used in constructors and member functions.
- Relies on built-in `new[]` / `delete[]` operators.


# Expat/MacOS Support/SmartPtr.h
## File Purpose
A template class implementing a basic scoped smart pointer for C++ that automatically deallocates the pointed-to object upon destruction. Provides pointer-like semantics (dereference, arrow operator) and explicit lifetime management methods.

## Core Responsibilities
- Wrap raw pointers and manage their lifetime
- Automatically delete pointed-to objects on destruction
- Provide operator overloads for pointer-like syntax (ΓåÆ, *, ())
- Support pointer reassignment with automatic cleanup of old pointer
- Offer explicit release/clear methods for manual ownership control
- Enable null-pointer checks via `present()`

## External Dependencies
- Standard C++: `new`, `delete` operators
- Expat library context: inferred from path; no explicit dependencies visible in this file

# Expat/sample/elements.cpp
## File Purpose
Sample application demonstrating the Expat XML parser library. Reads an XML document from a file (selected via cross-platform dialog) and outputs the element tree structure with indentation, attributes, and character data, along with error reporting on parse failures.

## Core Responsibilities
- Create and configure an XML parser with user-defined callbacks
- Display XML element hierarchy with tab indentation based on nesting depth
- Print element attributes prefixed with `$`
- Display text content between elements bracketed with `<<<` and `>>>`
- Handle file I/O with cross-platform open dialog support
- Report parse errors with line numbers
- Clean up parser resources on completion or error

## External Dependencies
- **Includes:** `stdio.h` (standard I/O), `stdlib.h` (memory, exit), `xmlparse.h` (Expat parser), `OpenSave_Interface.h` (cross-platform file dialog)
- **External symbols:** `XML_ParserCreate()`, `XML_SetUserData()`, `XML_SetCharacterDataHandler()`, `XML_SetElementHandler()`, `XML_Parse()`, `XML_ParserFree()`, `XML_ErrorString()`, `XML_GetErrorCode()`, `XML_GetCurrentLineNumber()`, `OpenFile()` (defined elsewhere; user-data callback parameters defined in xmlparse.h)

# Expat/xmlparse/hashtable.cpp
## File Purpose
Hash table implementation using linear probing with automatic resizing. Provides key-value storage for fast lookups (typically used by the Expat XML parser for entity and namespace resolution).

## Core Responsibilities
- Initialize and destroy hash table structures
- Look up entries by key with optional creation of new entries
- Automatically grow table when load factor exceeds 50%
- Iterate over populated entries in the table
- Perform string key comparison and hashing

## External Dependencies
- **Headers:** `xmldef.h` (for conditional memory management: stdlib.h, Windows API, or NSPR), `hashtable.h` (type definitions)
- **Memory:** malloc/calloc/free (may be redirected to HeapAlloc or NSPR depending on platform macros)
- **String:** Relies on null-terminated strings (KEY type)

# Expat/xmlparse/hashtable.h
## File Purpose
Header file defining the hash table API and data structures for the Expat XML parser. Provides lookup, initialization, destruction, and iteration operations over a simple hash table with unicode/ASCII key support.

## Core Responsibilities
- Define hash table data structure and key type (conditional on unicode configuration)
- Declare hash table lifecycle operations (init, destroy)
- Declare lookup/insert operation
- Provide iterator interface for traversing entries

## External Dependencies
- `<stddef.h>` ΓÇö for `size_t`
- Conditional compilation symbols: `XML_UNICODE`, `XML_UNICODE_WCHAR_T` (defined elsewhere)

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

## External Dependencies
- `xmldef.h`: Memory allocation macros (malloc, free, realloc), platform abstractions
- `xmlparse.h`: Public API types (XML_Parser, handler typedefs, error codes)
- `xmltok.h`: Encoding structures (ENCODING, INIT_ENCODING), token/state constants, encoding functions (XmlConvert, XmlEncode, etc.)
- `xmlrole.h`: XML grammar role/state machine (PROLOG_STATE, XmlPrologStateInit)
- `hashtable.h`: Hash table API (HASH_TABLE, lookup, hashTableInit, hashTableIterNext)
- Standard C: string.h, stdlib.h (via xmldef.h)

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

## External Dependencies
- `#include <stddef.h>` ΓÇô for `NULL` and `wchar_t` definition (only when `XML_UNICODE_WCHAR_T` defined)
- No external library dependencies; self-contained C API

# Expat/xmltok/asciitab.h
## File Purpose
Defines a static lookup table mapping ASCII byte values (0x00ΓÇô0x7F) to XML character type classifications. This table enables fast, single-lookup character categorization during XML tokenization without repeated conditional branching.

## Core Responsibilities
- Classifies ASCII bytes into XML-relevant categories (whitespace, digits, name characters, special symbols)
- Provides O(1) character type lookup by byte value as table index
- Supports XML tokenizer decisions (e.g., valid name-start characters, operators, delimiters)
- Handles ASCII control characters, spaces, line feeds, and carriage returns
- Distinguishes hex digits, name characters, and name-start characters for identifier parsing

## External Dependencies
- **Character type constants**: References undeclared `BT_*` macro/enum constants (e.g., `BT_NONXML`, `BT_S`, `BT_DIGIT`, `BT_NMSTRT`, `BT_HEX`, `BT_NAME`, `BT_EXCL`) ΓÇö defined elsewhere, likely in a shared header such as `xmltok.h` or an internal type definition.
- **License**: Mozilla Public License / GNU GPL dual-licensed.

# Expat/xmltok/dllmain.cpp
## File Purpose
Windows DLL (Dynamic Link Library) entry point for the Expat XML tokenizer library. Provides the required `DllMain` function that Windows calls when the DLL is loaded into or unloaded from a process. This is a minimal stub implementation with no custom initialization logic.

## Core Responsibilities
- Export the mandatory `DllMain` entry point required by Windows for DLL execution
- Signal successful DLL attachment/detachment to the operating system
- Comply with Windows DLL module initialization conventions

## External Dependencies
- `<windows.h>` ΓÇö Windows API header (HANDLE, BOOL, WINAPI, LPVOID types)
- Expat library context (inferred from file path)

---

**Note:** This file is from the Expat XML parser library, not a game engine. It is a standard Windows DLL stub with no game-specific or engine-specific logic.

# Expat/xmltok/iasciitab.h
## File Purpose

A lookup table that classifies ASCII byte values (0x00ΓÇô0x7F) into character type categories for XML tokenization. This variant treats carriage return (0xD) as whitespace (`BT_S`) rather than a distinct token type, diverging from the standard `asciitab.h`.

## Core Responsibilities

- Provides 256-entry character classification constants for the XML parser's tokenizer
- Maps each byte value to a `BT_*` token type (e.g., `BT_NMSTRT`, `BT_DIGIT`, `BT_S`, `BT_NONXML`)
- Distinguishes XML-valid name start characters, name continuation characters, digits, hex digits, and whitespace
- Handles special characters (delimiters: `<`, `>`, `=`, `/`, `&`, `%`, etc.)
- Designed to be included into tokenizer implementations via preprocessor directives

## External Dependencies

- **BT_* constants**: defined elsewhere (likely in `xmltok.h` or similar header); represents token/byte types:
  - `BT_NONXML`, `BT_S` (whitespace), `BT_LF` (linefeed), `BT_NAME`, `BT_NMSTRT` (name start), `BT_DIGIT`, `BT_HEX`
  - Special characters: `BT_LT`, `BT_GT`, `BT_EQUALS`, `BT_AMP`, `BT_PERCNT`, `BT_SEMI`, `BT_COLON`, etc.
- Included by tokenizer implementations (not a standalone header)

# Expat/xmltok/latin1tab.h
## File Purpose
A character classification lookup table for Latin-1 (ISO-8859-1) encoded XML parsing. Maps byte values (0x80ΓÇô0xFF) to character type constants indicating whether each byte can start or appear within XML names, or is merely "other" content.

## Core Responsibilities
- Provides byte-to-type mappings for the high byte range (0x80ΓÇô0xFF) in Latin-1 encoding
- Defines which bytes are valid XML name-start characters (`BT_NMSTRT`)
- Defines which bytes are valid XML name characters (`BT_NAME`)
- Categorizes remaining bytes as `BT_OTHER` (not name-related)
- Serves as an included constant data table for tokenizer/lexer logic

## External Dependencies
- **Undefined symbols**: `BT_NMSTRT`, `BT_NAME`, `BT_OTHER` ΓÇö likely enum/constant definitions from a header in the same or parent directory (e.g., `xmltok.h` or similar)
- **Licensing**: Dual MPL 1.1 / GPL license headers indicate this is part of the Expat XML parser project


# Expat/xmltok/nametab.h
## File Purpose
Static lookup table header for XML name character validation in the Expat parser. Provides bitmap and page-mapping tables to efficiently classify Unicode characters as valid XML name start/continuation characters.

## Core Responsibilities
- Define `namingBitmap[]`: Bit patterns encoding valid characters for XML identifiers across Unicode ranges
- Define `nmstrtPages[]`: Page indices mapping first-byte code points to bitmap sections for name-start characters
- Define `namePages[]`: Page indices mapping first-byte code points to bitmap sections for name-continuation characters
- Support rapid character classification without requiring full Unicode property lookups

## External Dependencies
- Defined elsewhere: consuming code that performs character lookups (tokenizer/validator in sibling `.c` files)
- No includes visible; pure data-only header


# Expat/xmltok/utf8tab.h
## File Purpose
A lookup table for UTF-8 byte classification used by Expat's XML tokenizer. Maps each possible byte value (0x00ΓÇô0xFF) to a byte type constant indicating whether it is a UTF-8 lead byte, continuation byte, invalid, or non-XML. This table enables efficient single-byte-lookup validation and parsing of UTF-8 encoded XML.

## Core Responsibilities
- Define or include a 256-entry byte classification lookup table
- Classify bytes as UTF-8 lead bytes (2, 3, or 4-byte sequences), continuation bytes, malformed bytes, or non-XML characters
- Enable O(1) UTF-8 validation during XML tokenization
- Support correct multi-byte UTF-8 character boundary detection

## External Dependencies
- `BT_TRAIL`, `BT_LEAD2`, `BT_LEAD3`, `BT_LEAD4`, `BT_NONXML`, `BT_MALFORM` ΓÇö byte type constants (defined elsewhere, likely in `xmltok.h` or similar)
- Expat XML tokenizer ΓÇö consumes this table for UTF-8 byte classification


# Expat/xmltok/xmldef.h
## File Purpose
A platform/environment abstraction header that provides conditional memory allocation macros and includes. It allows the Expat XML parser to work with different runtime environments (Windows heap manager, Mozilla NSPR, or standard C library) via compile-time configuration.

## Core Responsibilities
- Conditionally define memory allocation macros (malloc, calloc, free, realloc) based on target environment
- Include platform-specific headers (Windows.h, NSPR headers) when needed
- Provide a transparent abstraction layer so the rest of the codebase uses consistent malloc/free syntax regardless of the actual allocator
- Include string.h globally for string utilities

## External Dependencies
| Include | Condition | Purpose |
|---------|-----------|---------|
| `string.h` | Always | Standard string utilities |
| `windows.h` | If `XML_WINLIB` defined | Windows heap and API functions |
| `stdlib.h` | If `XML_WINLIB` not defined | Standard C memory/runtime |
| `nspr.h` | If `MOZILLA` defined | Mozilla NSPR memory allocation |

**Macro rewrites** (mallocs, callocs, etc.) conditionally redirect to platform-specific implementations via preprocessor substitution. Note: the `MOZILLA` block also redefines `int` to `int32` for type consistency.

# Expat/xmltok/xmlrole.cpp
## File Purpose
Implements an XML prolog state machine that classifies tokens into semantic roles (DOCTYPE, ENTITY, NOTATION, ATTLIST, ELEMENT declarations). Processes the portion of an XML document before the root element, validating structure and identifying declaration types and their components.

## Core Responsibilities
- State machine routing via function-pointer handlers for different prolog contexts
- Token-to-role classification (returns `XML_ROLE_*` constants)
- Grammar validation for DOCTYPE, ENTITY, NOTATION, ATTLIST, and ELEMENT declarations
- Tracking nesting depth for grouped content models (parentheses in ELEMENT declarations)
- Error state management and syntax error reporting

## External Dependencies
- `XmlNameMatchesAscii(enc, ptr, string)` ΓÇö Checks if token at `ptr` matches ASCII keyword (defined elsewhere)
- **Token constants:** `XML_TOK_*` (XML_TOK_NAME, XML_TOK_LITERAL, XML_TOK_DECL_OPEN, etc.) ΓÇö from tokenizer
- **Role constants:** `XML_ROLE_*` (XML_ROLE_DOCTYPE_NAME, XML_ROLE_ENTITY_VALUE, etc.) ΓÇö defined in xmlrole.h
- **Types:** `ENCODING` struct for character encoding info; `PROLOG_STATE` in xmlrole.h
- **Macros:** `MIN_BYTES_PER_CHAR(enc)` ΓÇö encoding-dependent byte width

# Expat/xmltok/xmlrole.h
## File Purpose
Defines XML structural role enumerations and prolog state management for the Expat tokenizer. Provides a state machine interface for categorizing tokens during XML prologue parsing and determining their semantic role (e.g., attribute name, element declaration, entity definition).

## Core Responsibilities
- Define enum constants for XML structural roles (XML_ROLE_* values) used during prologue parsing
- Define the PROLOG_STATE structure for tracking parser state during prologue processing
- Initialize and manage prologue parsing state
- Provide macro interface for invoking token role handlers

## External Dependencies
- **Includes:** `xmltok.h` (defines ENCODING struct, XML_TOK_* token constants, related macros)
- **Defined elsewhere:** ENCODING struct and token type constants used as inputs to role handlers

# Expat/xmltok/xmltok.cpp
## File Purpose
Core XML tokenizer and encoding infrastructure for the Expat parser. Implements character encoding detection, UTF-8/UTF-16 conversion, XML declaration parsing, and bootstraps encoding-specific tokenizer instances via template expansion.

## Core Responsibilities
- Encoding initialization and selection (UTF-8, UTF-16 LE/BE, ASCII, Latin-1, unknown)
- Automatic BOM detection and encoding auto-detection from byte patterns
- XML/text declaration parsing (version, encoding, standalone attributes)
- Character reference validation and encoding (UTF-8, UTF-16)
- Charset conversion between UTF-8, UTF-16, and single-byte encodings
- Instantiation of encoding-specific tokenizers via macro template expansion

## External Dependencies
- **Headers**: `xmldef.h` (memory/platform), `xmltok.h` (public API, ENCODING struct), `nametab.h` (character bitmaps)
- **Included sources (template expansion)**: `xmltok_impl.h` (BT_* enum), `xmltok_impl.c` (tokenizer state machines, included 3├ù with different PREFIX), `xmltok_ns.c` (namespace variant, included 2├ù)
- **Included tables**: `asciitab.h`, `utf8tab.h`, `latin1tab.h`, `iasciitab.h` (character type arrays)
- **Symbols defined elsewhere**: XML_TOK_* constants, ENCODING struct members, BT_* byte type enums, PREFIX/VTABLE macros for code generation

# Expat/xmltok/xmltok.h
## File Purpose
Public header for Expat's XML tokenizer module. Defines token type constants, encoding abstraction structures, and tokenization entry points (via macros) that dispatch to encoding-specific scanner functions.

## Core Responsibilities
- Enumerate XML token types (tags, attributes, CDATA, processing instructions, etc.)
- Define parser state constants (prolog, content, CDATA section)
- Provide `ENCODING` structure abstracting encoding-specific operations via function pointers
- Declare tokenization macros that dispatch to state-specific scanners
- Declare encoding initialization and character encoding conversion functions
- Track line/column position during parsing via `POSITION` struct

## External Dependencies
- No explicit includes visible
- Expat encoding implementations (referenced but not defined in this file)
- C standard library (implicit from function signatures)

# Expat/xmltok/xmltok_impl.cpp
## File Purpose
Core XML tokenization implementation for the Expat parser. Provides encoding-agnostic scanning functions that recognize and validate XML constructs (comments, tags, attributes, references, CDATA sections, etc.) by sequentially examining bytes and returning token types.

## Core Responsibilities
- Scan and validate XML markup structures (comments, processing instructions, CDATA sections, tags, attributes)
- Parse entity and character references (both decimal and hexadecimal)
- Tokenize attribute values, entity values, and literal strings
- Validate XML name syntax (element names, attribute names, entity references)
- Track source position (line/column numbers) for error reporting
- Support multiple character encodings through parameterized macro expansion

## External Dependencies
- **Includes/Macros:** PREFIX (generates encoding-specific function names)
- **Encoding macros:** MINBPC (minimum bytes per character), BYTE_TYPE, CHAR_MATCHES, BYTE_TO_ASCII, IS_INVALID_CHAR, IS_NAME_CHAR, IS_NMSTRT_CHAR
- **Token constants:** XML_TOK_COMMENT, XML_TOK_INVALID, XML_TOK_PARTIAL, XML_TOK_START_TAG_WITH_ATTS, etc.
- **Byte type constants:** BT_MINUS, BT_DIGIT, BT_NMSTRT, BT_NONASCII, BT_CR, BT_LF, etc. (defined elsewhere)
- **External symbols:** checkCharRefNumber (validates code points), POSITION, ATTRIBUTE, ENCODING (defined in headers)

# Expat/xmltok/xmltok_impl.h
## File Purpose
Defines byte-type classification constants used by the XML tokenizer to categorize characters and byte sequences. This is a foundational enum for XML character parsing, enabling efficient character-by-character analysis during tokenization.

## Core Responsibilities
- Enumerate byte type categories for XML character classification
- Distinguish between syntactically significant XML characters (delimiters, operators)
- Identify character class properties (whitespace, name starts, digits, UTF-8 leads/trails)
- Provide mnemonics for XML tokenizer logic to consume and classify input

## External Dependencies
- `<stddef.h>` ΓÇö standard C definitions (included at end of file)

**Notes on enum constants:**
- **Markup delimiters:** `BT_LT` (`<`), `BT_GT` (`>`), `BT_SOL` (`/`), `BT_EXCL` (`!`), `BT_QUEST` (`?`), `BT_RSQB` (`]`), `BT_LSQB` (`[`)
- **Entity/name markers:** `BT_AMP` (`&`), `BT_SEMI` (`;`), `BT_NUM` (`#`), `BT_PERCNT` (`%`)
- **Operators & syntax:** `BT_EQUALS` (`=`), `BT_QUOT` (`"`), `BT_APOS` (`'`), `BT_MINUS` (`-`), `BT_COLON` (`:`)
- **Character class:** `BT_NMSTRT` (name start), `BT_NAME` (name char), `BT_DIGIT`, `BT_HEX`, `BT_S` (whitespace), `BT_CR`, `BT_LF`
- **UTF-8:** `BT_LEAD2`, `BT_LEAD3`, `BT_LEAD4` (leading bytes), `BT_TRAIL` (continuation byte)
- **Catch-all:** `BT_NONXML`, `BT_MALFORM`, `BT_OTHER`, `BT_NONASCII`

# Expat/xmltok/xmltok_ns.cpp
## File Purpose
Provides namespace-scoped XML encoding initialization and detection for the Expat parser. Handles internal encoding references (UTF-8, UTF-16 with endianness detection), encoding array management, and XML declaration parsing to extract encoding directives.

## Core Responsibilities
- Return internal UTF-8 and UTF-16 encoding references with endianness awareness
- Maintain a static array of supported character encodings
- Initialize `INIT_ENCODING` structures with encoding-specific scanner callbacks
- Detect encoding declarations from XML prolog text
- Parse XML declarations to extract version, encoding name, and standalone flag

## External Dependencies
- **Macro wrapper:** `NS()` (likely preprocessor namespace mangling, pattern suggests C-style namespacing).
- **Functions defined elsewhere:** `initScan`, `getEncodingIndex`, `XmlUtf8Convert`, `streqci`, `doParseXmlDecl`, `initUpdatePosition`.
- **Types defined elsewhere:** `ENCODING`, `INIT_ENCODING`.
- **Constants:** `XML_BYTE_ORDER`, `XML_PROLOG_STATE`, `XML_CONTENT_STATE`, `UNKNOWN_ENC`, `ENCODING_MAX`.

# Expat/xmlwf/codepage.cpp
## File Purpose
Windows codepage conversion utility for the xmlwf XML parser tool. Builds Unicode conversion tables for multibyte codepages and converts individual multibyte sequences to Unicode characters. Platform-specific: full implementation on Windows, stub implementations on other platforms.

## Core Responsibilities
- Build a 256-entry lookup table mapping byte values to Unicode codepoints for a specified codepage
- Convert 2-byte multibyte character sequences to Unicode wide characters
- Identify and mark lead bytes (first byte of multibyte sequences) in codepage mappings
- Validate codepage support via Windows CPINFO structure
- Provide cross-platform interface (functional on Windows, no-ops elsewhere)

## External Dependencies
- `<windows.h>` ΓÇö Windows API (GetCPInfo, MultiByteToWideChar); conditional compilation on `WIN32`
- `codepage.h` ΓÇö Header declaring both functions
- **Defined elsewhere**: CPINFO structure, GetCPInfo, MultiByteToWideChar (Windows API)

# Expat/xmlwf/codepage.h
## File Purpose
Header file declaring the public interface for code page mapping and character conversion utilities. Provides functions to query code page capabilities and convert characters from specific code pages.

## Core Responsibilities
- Declare code page mapping interface to retrieve character mappings for a given code page
- Declare code page conversion interface to convert individual characters from a specified code page
- Establish contracts for code page support in the `xmlwf` tool

## External Dependencies
- Standard C library (inferred from `int` and pointer types; no explicit includes in this header)

# Expat/xmlwf/filemap.h
## File Purpose
Header file declaring the `filemap` function interface for the Expat XML parser utility. Provides a mechanism to read files and invoke a callback processor on the file contents, with support for both Unicode and ASCII variants.

## Core Responsibilities
- Declare the `filemap` function interface
- Support both wide-character (Unicode) and byte-character (ASCII) file processing
- Define the callback processor function signature for consuming file data
- Abstract file I/O and memory mapping details from callers

## External Dependencies
- `<stddef.h>` ΓÇö for `size_t` and standard type definitions
- `XML_UNICODE` preprocessor symbol ΓÇö controls which variant is compiled (wide-char vs. byte-char)

# Expat/xmlwf/readfilemap.cpp
## File Purpose
Utility function that reads an entire file from disk into memory and passes its contents to a callback processor. Part of xmlwf (XML Well-Formedness checker) tool infrastructure. Handles file I/O with cross-platform binary mode support and error reporting.

## Core Responsibilities
- Open files in binary read-only mode with POSIX system calls
- Validate files are regular files (not directories or devices)
- Allocate sufficient heap memory for entire file contents
- Read file data into memory with integrity validation
- Invoke caller-provided callback processor with file data
- Release resources (memory, file descriptor) and report status
- Report errors to stderr with context information

## External Dependencies
- **POSIX system calls:** `open()`, `fstat()`, `read()`, `close()`
- **Standard C library:** `malloc()`, `free()`, `stdio` functions (`perror()`, `fprintf()`)
- **Platform abstraction:** `O_BINARY` macro defined locally to handle Windows binary mode

# Expat/xmlwf/unixfilemap.cpp
## File Purpose
Unix-specific file mapping utility for the Expat XML parser's `xmlwf` tool. Provides a cross-platform abstraction for reading file contents via memory mapping, using POSIX `mmap()` on Unix systems to efficiently load XML files for processing.

## Core Responsibilities
- Open and validate regular files for reading
- Memory-map file contents into process address space using POSIX `mmap()`
- Invoke a user-provided processor callback with mapped buffer and file metadata
- Perform resource cleanup (unmap memory, close file descriptor)
- Report errors via `perror()` and `fprintf()`

## External Dependencies
- **System headers:** `<sys/types.h>`, `<sys/mman.h>`, `<sys/stat.h>`, `<fcntl.h>`, `<errno.h>`, `<string.h>`, `<stdio.h>`
  - POSIX file operations, memory mapping, stat/mode checking
- **Bundled header:** `filemap.h`
  - Declares `filemap()` signature; conditionally wide-char variant if `XML_UNICODE` is defined
- **Implicit external:** `processor` callback (defined by caller, typically XML parser)

# Expat/xmlwf/win32filemap.cpp
## File Purpose
Provides Windows-specific file mapping functionality for the Expat XML parser. Maps files into memory via Windows memory-mapped file APIs, then invokes a processor callback on the mapped data for efficient file handling without explicit read operations.

## Core Responsibilities
- Open files using Windows `CreateFile()` API with sequential scan hints
- Validate file sizes (reject >2GB files)
- Create and manage memory-mapped file objects
- Map file views into process virtual address space
- Invoke processor callbacks on mapped data
- Clean up system resources (views, mappings, file handles)
- Format and report Windows system errors to stderr

## External Dependencies
- **System headers:** `<windows.h>`, `<stdio.h>`, `<tchar.h>`
- **Filemap interface:** `"filemap.h"` (defines `filemap()` signature)
- **Windows API functions:** `CreateFile()`, `GetFileSize()`, `CreateFileMapping()`, `MapViewOfFile()`, `UnmapViewOfFile()`, `CloseHandle()`, `FormatMessage()`, `GetLastError()`, `LocalFree()`, `MAKELANGID()`
- **Character encoding:** Conditional support for Unicode (`XML_UNICODE_WCHAR_T` ΓåÆ `_UNICODE`); uses `TCHAR` and `_ftprintf()` for portability

# Expat/xmlwf/xmlfile.cpp
## File Purpose
Implements XML file processing for the Expat parser with two modes: memory-mapped file reading and stream-based I/O. Handles external entity references via separate parser instances and provides error reporting with source location information.

## Core Responsibilities
- Parse XML files from disk using Expat's XML_Parser
- Support memory-mapped and stream-based processing modes (flag-driven)
- Create and manage child parsers for external entity resolution
- Resolve relative file paths to absolute paths for external entities
- Report parse errors with line/column numbers
- Read files in configurable chunks (8KB production, 16 bytes debug)

## External Dependencies
- **Parser API**: XML_Parser, XML_Parse, XML_ParseBuffer, XML_GetBuffer, XML_GetErrorCode, XML_ErrorString, XML_GetErrorLineNumber/ColumnNumber, XML_SetBase, XML_ExternalEntityParserCreate, XML_ParserFree, XML_SetExternalEntityRefHandler
- **File I/O**: topen (open), read, close (platform-abstracted via xmltchar.h macros)
- **Memory**: malloc, free
- **String ops**: tcslen, tcscpy, tcsrchr (platform-abstracted via xmltchar.h)
- **I/O**: filemap (defined elsewhere; memory-maps and processes files), ftprintf (formatted output)

# Expat/xmlwf/xmlfile.h
## File Purpose
Header file for the xmlwf (XML well-formedness checker) utility. Declares the public interface for processing XML files and defines flags that control file processing behavior.

## Core Responsibilities
- Define flag constants for controlling XML file processing behavior (`XML_MAP_FILE`, `XML_EXTERNAL_ENTITIES`)
- Declare the `XML_ProcessFile` function, the primary entry point for validating/parsing XML files
- Provide dual licensing notices (MPL 1.1 and GPL compatibility)

## External Dependencies
- **Type:** `XML_Parser` ΓÇö opaque parser handle (defined elsewhere in expat library; initialized by caller)
- **Type:** `XML_Char` ΓÇö character type abstraction for expat (supports both UTF-8 and wide-char builds)
- **Library:** Core expat XML parsing library (not shown in this file)

# Expat/xmlwf/xmltchar.h
## File Purpose
Provides compile-time character type abstraction for the xmlwf utility, enabling it to operate with either UTF-16 Unicode (wide-character) or ASCII/multibyte string representations. Uses conditional compilation to select the appropriate C standard library functions and macros based on the `XML_UNICODE` configuration flag.

## Core Responsibilities
- Abstracts string operations (comparison, copying, length, search) behind unified macro names
- Abstracts I/O operations (formatted print, file open, character output) behind unified macro names  
- Provides character literal abstraction via the `T()` macro to support both `"string"` and `L"string"` literals
- Validates that Unicode mode requires 16-bit `wchar_t` compatibility
- Allows the rest of xmlwf code to remain independent of character encoding choice

## External Dependencies
- **Wide-character API** (used when `XML_UNICODE` is defined): `fwprintf`, `_wfopen`, `fputws`, `putwc`, `wcscmp`, `wcscpy`, `wcscat`, `wcschr`, `wcsrchr`, `wcslen`, `_wperror`, `_wopen`, `wmain`, `_wremove`
- **Standard C API** (used when `XML_UNICODE` is not defined): `fprintf`, `fopen`, `fputs`, `putc`, `strcmp`, `strcpy`, `strcat`, `strchr`, `strrchr`, `strlen`, `perror`, `open`, `main`, `remove`
- **Preprocessor**: `#ifdef`, `#else`, `#endif`, `#error`, `##` (token pasting)

# Expat/xmlwf/xmlwf.cpp
## File Purpose
Main entry point for `xmlwf`, a command-line XML validation and reformatting utility. Parses XML documents using the Expat parser library and outputs them in various formats (canonical XML, namespace-expanded, metadata, or timing-only modes). Supports multiple character encodings including Windows code pages.

## Core Responsibilities
- Parse command-line arguments and configure parser behavior (encoding, output mode, namespace handling, external entities)
- Create and configure XML_Parser instances with appropriate callback handlers
- Register handlers for XML parsing events (elements, character data, processing instructions, comments, CDATA, declarations)
- Output parsed XML in configurable formats: default (namespace-unaware), namespace-expanded (NS), compact markup (C), or metadata (M)
- Implement character escaping and attribute sorting for deterministic output
- Handle unknown character encodings (Windows code pages) via custom encoding handlers
- Manage file I/O and output directory organization

## External Dependencies
- **xmlparse.h:** Core Expat API (parser creation, handler registration, parsing, error codes).
- **codepage.h:** Windows code page mapping (`codepageMap()`, `codepageConvert()`)
- **xmlfile.h:** File processing constants (`XML_MAP_FILE`, `XML_EXTERNAL_ENTITIES`) and `XML_ProcessFile()` declaration.
- **xmltchar.h:** Character type abstractions (`T()`, `tcslen()`, `tcscpy()`, `ftprintf()`, etc.) supporting both ANSI and Unicode builds.
- **Standard C:** `stdio.h`, `stdlib.h`, `string.h` (I/O, memory, string ops).
- **MSVC-specific:** `crtdbg.h` (debug memory tracking on Windows).


