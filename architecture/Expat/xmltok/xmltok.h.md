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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `POSITION` | struct | Tracks current line and column during parsing (0-indexed) |
| `ATTRIBUTE` | struct | Represents an XML attribute with name and value range pointers |
| `ENCODING` | struct | Function pointer table for encoding-specific tokenization, name matching, and conversion operations |
| `INIT_ENCODING` | struct | Wrapper holding initial encoding state and pointer to resolved encoding |

## Global / File-Static State
None.

## Key Functions / Methods

### XmlTok / XmlPrologTok / XmlContentTok / XmlCdataSectionTok
- **Signature:** Macros dispatching via `(enc)->scanners[state](enc, ptr, end, nextTokPtr)`
- **Purpose:** Scan XML input from `ptr` to `end`, returning next token type and advancing `nextTokPtr`
- **Inputs:** Encoding struct, state (0=prolog, 1=content, 2=CDATA), input range `[ptr, end)`
- **Outputs/Return:** Token type constant (negative=error/incomplete, 0+=valid token); `nextTokPtr` set to next position
- **Side effects:** None (read-only scanning)
- **Notes:** Returns `XML_TOK_NONE` if `ptr==end`; `XML_TOK_PARTIAL` if token incomplete; `XML_TOK_INVALID` on invalid sequence with `nextTokPtr` pointing to offending char

### XmlLiteralTok / XmlAttributeValueTok / XmlEntityValueTok
- **Signature:** Macros dispatching via `(enc)->literalScanners[literalType](enc, ptr, end, nextTokPtr)`
- **Purpose:** Second-level tokenization of literal content already returned by `XmlTok`
- **Inputs:** Encoding, literal type (0=attribute value, 1=entity value), input range
- **Outputs/Return:** Token type for literal content

### XmlParseXmlDecl / XmlParseXmlDeclNS
- **Signature:** `int (int isGeneralTextEntity, const ENCODING *enc, const char *ptr, const char *end, const char **badPtr, const char **versionPtr, const char **encodingNamePtr, const ENCODING **namedEncodingPtr, int *standalonePtr)`
- **Purpose:** Parse XML declaration and set encoding/version/standalone attributes
- **Outputs/Return:** Integer status; pointers filled with parsed declaration components

### XmlInitEncoding / XmlInitEncodingNS
- **Signature:** `int (INIT_ENCODING *, const ENCODING **, const char *name)`
- **Purpose:** Initialize encoding from name string; populate encoding function table

### XmlUtf8Encode / XmlUtf16Encode
- **Signature:** `int (int charNumber, char/unsigned short *buf)`
- **Purpose:** Encode Unicode code point into UTF-8 or UTF-16
- **Outputs/Return:** Number of bytes/shorts written; caller must provide buffer of size ΓëÑ4 (UTF-8) or ΓëÑ2 (UTF-16)

### Helper Macros
- `XmlSameName`, `XmlNameMatchesAscii`, `XmlNameLength`: Name comparison/length operations dispatched via encoding
- `XmlSkipS`, `XmlGetAttributes`, `XmlCharRefNumber`: Whitespace skipping, attribute extraction, character reference decoding
- `XmlUpdatePosition`, `XmlIsPublicId`: Position tracking and PUBLIC identifier validation

## Control Flow Notes
Tokenization is **state-driven** via three XML parsing contexts:
- **XML_PROLOG_STATE** (0): Parsing XML declaration, DTD, comments
- **XML_CONTENT_STATE** (1): Parsing element content (tags, text, CDATA sections)
- **XML_CDATA_SECTION_STATE** (2): Inside `<![CDATA[...]]>` blocks

Each state has its own scanner function in the encoding's `scanners[]` array. Token type ranges from negative (errors: `XML_TOK_PARTIAL_CHAR` to `XML_TOK_TRAILING_RSQB`) to non-negative valid tokens (0=invalid base, 1+=specific token types).

## External Dependencies
- No explicit includes visible
- Expat encoding implementations (referenced but not defined in this file)
- C standard library (implicit from function signatures)
