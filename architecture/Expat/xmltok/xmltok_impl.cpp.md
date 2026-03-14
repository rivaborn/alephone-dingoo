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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ENCODING | struct/opaque | Character encoding metadata; passed to all scanning functions |
| ATTRIBUTE | struct | Stores name, value pointer, and value end for a single XML attribute |
| POSITION | struct | Tracks line number and column number during parsing |

## Global / File-Static State
None.

## Key Functions / Methods

### scanComment
- **Signature:** `static int PREFIX(scanComment)(ENCODING *enc, const char *ptr, const char *end, const char **nextTokPtr)`
- **Purpose:** Scan XML comment content (`<!-- ... -->`). Expects ptr to point after `<!-`.
- **Inputs:** enc (encoding), ptr (position after `<!-`), end (buffer end), nextTokPtr (output)
- **Outputs/Return:** Token type (XML_TOK_COMMENT, XML_TOK_INVALID, XML_TOK_PARTIAL); sets nextTokPtr to next unprocessed position
- **Side effects:** Reads buffer; may modify nextTokPtr
- **Calls:** BYTE_TYPE, CHAR_MATCHES, MINBPC (macros)
- **Notes:** Validates comment does not contain `--` except before `>`. Returns PARTIAL if buffer ends mid-comment.

### scanLt
- **Signature:** `static int PREFIX(scanLt)(const ENCODING *enc, const char *ptr, const char *end, const char **nextTokPtr)`
- **Purpose:** Dispatch scanner after seeing `<`. Determines token type (comment, CDATA, PI, end-tag, start-tag, or invalid).
- **Inputs:** enc, ptr (after `<`), end, nextTokPtr
- **Outputs/Return:** Token type or dispatches to sub-scanners
- **Calls:** scanComment, scanCdataSection, scanPi, scanEndTag, scanAtts
- **Notes:** Central dispatcher for post-`<` content; handles XML namespace colon syntax conditionally.

### scanAtts
- **Signature:** `static int PREFIX(scanAtts)(const ENCODING *enc, const char *ptr, const char *end, const char **nextTokPtr)`
- **Purpose:** Scan and validate attribute name-value pairs in a start tag. Expects ptr at first attribute name character.
- **Inputs:** enc, ptr (at first char of attribute name), end, nextTokPtr
- **Outputs/Return:** Token type (XML_TOK_START_TAG_WITH_ATTS, XML_TOK_EMPTY_ELEMENT_WITH_ATTS, etc.)
- **Side effects:** Parses quoted attribute values; calls scanRef for entity references
- **Notes:** Handles namespace colons; validates attribute syntax; detects normalization issues.

### charRefNumber
- **Signature:** `static int PREFIX(charRefNumber)(const ENCODING *enc, const char *ptr)`
- **Purpose:** Extract and convert numeric character reference (e.g., `&#123;` or `&#xAB;`) to integer code point.
- **Inputs:** enc, ptr (at character after `&#` or `&#x`)
- **Outputs/Return:** Integer code point; -1 if invalid or out of range
- **Notes:** Handles both decimal and hex formats; validates against 0x110000 upper bound.

### getAtts
- **Signature:** `static int PREFIX(getAtts)(const ENCODING *enc, const char *ptr, int attsMax, ATTRIBUTE *atts)`
- **Purpose:** Extract attribute name/value pointers from a well-formed start/empty-element tag.
- **Inputs:** enc, ptr (at tag name), attsMax (capacity), atts (output array)
- **Outputs/Return:** Count of attributes found
- **Side effects:** Fills atts array with attribute metadata
- **Notes:** Called only on well-formed tags (post-validation); also detects attribute value normalization state.

### updatePosition
- **Signature:** `static void PREFIX(updatePosition)(const ENCODING *enc, const char *ptr, const char *end, POSITION *pos)`
- **Purpose:** Advance line and column counters based on characters scanned.
- **Inputs:** enc, ptr (start), end (boundary), pos (position state)
- **Side effects:** Modifies pos->lineNumber and pos->columnNumber
- **Notes:** Recognizes CR, LF, and CR/LF pairs; multibye-char aware.

## Control Flow Notes
This file is part of the Expat tokenization pipeline. Functions are called in a dispatching pattern: a top-level parser calls `scanLt`, which branches to appropriate sub-scanners (comments, PIs, tags, etc.). Scanning proceeds character-by-character using BYTE_TYPE classification. Token types (XML_TOK_*) are returned to caller, which may then call functions like `getAtts` or `charRefNumber` to extract semantic content. The file is included via macro expansion to generate encoding-specific versions.

## External Dependencies
- **Includes/Macros:** PREFIX (generates encoding-specific function names)
- **Encoding macros:** MINBPC (minimum bytes per character), BYTE_TYPE, CHAR_MATCHES, BYTE_TO_ASCII, IS_INVALID_CHAR, IS_NAME_CHAR, IS_NMSTRT_CHAR
- **Token constants:** XML_TOK_COMMENT, XML_TOK_INVALID, XML_TOK_PARTIAL, XML_TOK_START_TAG_WITH_ATTS, etc.
- **Byte type constants:** BT_MINUS, BT_DIGIT, BT_NMSTRT, BT_NONASCII, BT_CR, BT_LF, etc. (defined elsewhere)
- **External symbols:** checkCharRefNumber (validates code points), POSITION, ATTRIBUTE, ENCODING (defined in headers)
