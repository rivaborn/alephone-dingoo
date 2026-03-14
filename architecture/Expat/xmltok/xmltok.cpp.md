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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `struct normal_encoding` | struct | Wraps ENCODING core with character type table (256 bytes) and encoding-specific function pointers for name/nmstart/invalid validation |
| `struct unknown_encoding` | struct | Extends normal_encoding with user-provided conversion callback and precomputed UTF-16/UTF-8 lookup tables for unknown charsets |
| `ENCODING` | typedef (from xmltok.h) | Virtual interface: scanner functions for prolog/content/CDATA states, literal scanners, and conversion callbacks (utf8Convert, utf16Convert) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `utf8_encoding` | `const struct normal_encoding` | static | Predefined UTF-8 encoding instance with BOM handling |
| `utf8_encoding_ns` | `const struct normal_encoding` | static | UTF-8 encoding with namespace processing (BT_COLON) |
| `internal_utf8_encoding` | `const struct normal_encoding` | static | Internal variant using different ASCII table |
| `latin1_encoding` | `const struct normal_encoding` | static | Predefined Latin-1 (ISO-8859-1) encoding |
| `ascii_encoding` | `const struct normal_encoding` | static | Predefined ASCII encoding (pass-through) |
| `little2_encoding` / `big2_encoding` | `const struct normal_encoding` | static | UTF-16 LE/BE encodings with endian-specific byte handlers |
| `namingBitmap`, `nmstrtPages`, `namePages` | imported from nametab.h | static | Unicode character classification bitmaps for name/nmstart validation |

## Key Functions / Methods

### XmlUtf8Encode
- Signature: `int XmlUtf8Encode(int c, char *buf)`
- Purpose: Encode a single Unicode code point to UTF-8 bytes (1ΓÇô4 bytes).
- Inputs: Code point (c), output buffer (buf, min 4 bytes)
- Outputs/Return: Number of bytes written (1ΓÇô4), or 0 if code point is invalid
- Side effects: Writes to output buffer
- Calls: None
- Notes: Validates minimum-length encoding; rejects negative values and code points ΓëÑ 0x110000

### XmlUtf16Encode
- Signature: `int XmlUtf16Encode(int charNum, unsigned short *buf)`
- Purpose: Encode Unicode code point to UTF-16 (optionally as surrogate pair for > U+FFFF).
- Inputs: Code point, output buffer (min 2 shorts)
- Outputs/Return: 1 or 2 (for surrogate pair), 0 if invalid
- Side effects: Writes to buffer
- Calls: None
- Notes: Characters ΓëÑ 0x10000 encoded as surrogate pair; rejects negatives and ΓëÑ 0x110000

### XmlInitUnknownEncoding
- Signature: `ENCODING * XmlInitUnknownEncoding(void *mem, int *table, int (*convert)(void *userData, const char *p), void *userData)`
- Purpose: Initialize encoding from user-supplied byteΓåÆcodepoint mapping table for unknown charsets.
- Inputs: Preallocated memory, 256-entry lookup table, conversion callback, callback user data
- Outputs/Return: Pointer to initialized ENCODING in mem, or NULL on validation failure
- Side effects: Populates memory struct; validates table consistency; pre-encodes UTF-8/UTF-16 tables
- Calls: `XmlUtf8Encode`, `UCS2_GET_NAMING` (macro for bitmap lookup), `checkCharRefNumber`
- Notes: Validates bytes 0ΓÇô127 match Latin-1; rejects duplicate/out-of-range mappings; marks multibyte sequences via negative values

### utf8_toUtf8
- Signature: `static void utf8_toUtf8(const ENCODING *enc, const char **fromP, const char *fromLim, char **toP, const char *toLim)`
- Purpose: Copy UTF-8 bytes from source to destination without splitting multibyte chars at boundaries.
- Inputs: Source buffer range (fromP/fromLim), destination buffer (toP/toLim)
- Outputs/Return: Updates fromP and toP pointers
- Side effects: Copies bytes; rewinds fromLim to avoid partial char at end
- Calls: None
- Notes: Checks byte 0xC0 mask (0xC0) to detect continuation bytes and back up if needed

### utf8_toUtf16
- Signature: `static void utf8_toUtf16(const ENCODING *enc, const char **fromP, const char *fromLim, unsigned short **toP, const unsigned short *toLim)`
- Purpose: Decode UTF-8 to UTF-16, handling 2/3/4-byte sequences and surrogates for astral planes.
- Inputs: UTF-8 source (fromP/fromLim), UTF-16 destination (toP/toLim), encoding context
- Outputs/Return: Updates source/destination pointers; writes UTF-16 code units
- Side effects: Decodes and writes to destination; may use 2 output slots for 4-byte input
- Calls: `BYTE_TYPE` macro to classify UTF-8 lead bytes
- Notes: 4-byte sequences (BT_LEAD4) generate surrogate pairs; stops early if insufficient output space

### latin1_toUtf8 / latin1_toUtf16
- Signature: `static void latin1_toUtf8/16(const ENCODING *enc, const char **fromP, const char *fromLim, char/**unsigned short **toP, const char/*unsigned short *toLim)`
- Purpose: Convert Latin-1 (ISO-8859-1) to UTF-8 or UTF-16.
- Inputs: Latin-1 source, target encoding destination
- Outputs/Return: Converted bytes/shorts written to destination; updates pointers
- Side effects: Byte transformation; Latin-1 high bytes (ΓëÑ 0x80) encoded as 2-byte UTF-8
- Calls: None
- Notes: Direct mapping; no validation needed

### ascii_toUtf8
- Signature: `static void ascii_toUtf8(const ENCODING *enc, const char **fromP, const char *fromLim, char **toP, const char *toLim)`
- Purpose: ASCII to UTF-8 conversion (pass-through; ASCII is valid UTF-8).
- Inputs: ASCII source, UTF-8 destination
- Outputs/Return: Copied bytes; updates pointers
- Side effects: Byte copy
- Calls: None

### doParseXmlDecl
- Signature: `static int doParseXmlDecl(const ENCODING *(*encodingFinder)(...), int isGeneralTextEntity, const ENCODING *enc, const char *ptr, const char *end, const char **badPtr, const char **versionPtr, const char **encodingName, const ENCODING **encoding, int *standalone)`
- Purpose: Parse XML/text declaration (<?xml ... ?>) to extract version, encoding, standalone attributes.
- Inputs: Encoding context, buffer (ptr/end), text entity flag, output pointers for parsed values
- Outputs/Return: 1 on success, 0 on parse error; sets badPtr on failure; populates version/encoding/standalone pointers
- Side effects: None (read-only parsing); may invoke encodingFinder callback
- Calls: `parsePseudoAttribute`, `XmlNameMatchesAscii`, `encodingFinder` callback
- Notes: Text decls require encoding; doc decls require version; standalone must be "yes" or "no"

### initScan
- Signature: `static int initScan(const ENCODING **encodingTable, const INIT_ENCODING *enc, int state, const char *ptr, const char *end, const char **nextTokPtr)`
- Purpose: Detect XML encoding from BOM and first bytes; initialize encoding for main parser.
- Inputs: Encoding table (indexed by enum), initial encoding, parser state, buffer
- Outputs/Return: Token type (XML_TOK_BOM, XML_TOK_PARTIAL, or result of XmlTok); sets *encPtr to detected encoding
- Side effects: Selects and sets encoding; updates nextTokPtr
- Calls: `XmlTok` recursively with detected encoding
- Notes: Detects UTF-16 BOM (0xFEFF / 0xFFFE), UTF-8 BOM (0xEFBBBF), UTF-16 via null byte patterns (0x00, 0x3C bytes)

### checkCharRefNumber
- Signature: `static int checkCharRefNumber(int result)`
- Purpose: Validate parsed character reference against XML rules (reject surrogates, forbidden code points).
- Inputs: Decoded code point
- Outputs/Return: Original value if valid, ΓêÆ1 if invalid
- Side effects: None
- Calls: None
- Notes: Rejects U+D800ΓÇôU+DFFF (surrogates), U+FFFE, U+FFFF, non-XML bytes in Latin-1 range

### getEncodingIndex
- Signature: `static int getEncodingIndex(const char *name)`
- Purpose: Map encoding name (C string) to internal enum index.
- Inputs: Encoding name ("ISO-8859-1", "UTF-8", etc.) or NULL
- Outputs/Return: Enum value (0ΓÇô6), UNKNOWN_ENC, or NO_ENC
- Side effects: None
- Calls: `streqci`
- Notes: Case-insensitive; matches 6 standard encodings

### streqci
- Signature: `static int streqci(const char *s1, const char *s2)`
- Purpose: Case-insensitive C string comparison.
- Inputs: Two strings
- Outputs/Return: 1 if equal (case-insensitive), 0 otherwise
- Side effects: None
- Calls: None

**Utility Functions (brief)**:
- `toAscii` ΓÇô Decode char to ASCII; returns ΓêÆ1 if invalid or multibyte
- `isSpace` ΓÇô Test for XML whitespace (space, tab, CR, LF)
- `parsePseudoAttribute` ΓÇô Parse name=value in XML declarations
- `unknown_isName`, `unknown_isNmstrt`, `unknown_isInvalid` ΓÇô Character validation for unknown encodings via callback
- `unknown_toUtf8`, `unknown_toUtf16` ΓÇô Conversion for unknown encodings

## Control Flow Notes

1. **Startup**: `initScan` detects encoding from BOM/bytes; selects from encoding table.
2. **Declaration**: `doParseXmlDecl` parses <?xml...?> and may trigger encoding switch.
3. **Tokenization**: Included `xmltok_impl.c` (via PREFIX macro, 3├ù for normal/little2/big2) implements state machines for prolog/content/CDATA. Function pointers in ENCODING struct dispatch to encoding-specific handlers (e.g., utf8_isName2).
4. **Conversion**: Output/buffer operations call utf8Convert or utf16Convert callbacks as needed.

## External Dependencies
- **Headers**: `xmldef.h` (memory/platform), `xmltok.h` (public API, ENCODING struct), `nametab.h` (character bitmaps)
- **Included sources (template expansion)**: `xmltok_impl.h` (BT_* enum), `xmltok_impl.c` (tokenizer state machines, included 3├ù with different PREFIX), `xmltok_ns.c` (namespace variant, included 2├ù)
- **Included tables**: `asciitab.h`, `utf8tab.h`, `latin1tab.h`, `iasciitab.h` (character type arrays)
- **Symbols defined elsewhere**: XML_TOK_* constants, ENCODING struct members, BT_* byte type enums, PREFIX/VTABLE macros for code generation
