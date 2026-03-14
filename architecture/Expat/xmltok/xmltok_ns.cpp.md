# Expat/xmltok/xmltok_ns.cpp

## File Purpose
Provides namespace-scoped XML encoding initialization and detection for the Expat parser. Handles internal encoding references (UTF-8, UTF-16 with endianness detection), encoding array management, and XML declaration parsing to extract encoding directives.

## Core Responsibilities
- Return internal UTF-8 and UTF-16 encoding references with endianness awareness
- Maintain a static array of supported character encodings
- Initialize `INIT_ENCODING` structures with encoding-specific scanner callbacks
- Detect encoding declarations from XML prolog text
- Parse XML declarations to extract version, encoding name, and standalone flag

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ENCODING | struct | Character encoding descriptor (defined elsewhere) |
| INIT_ENCODING | struct | Mutable initialization wrapper with scanner state machines |
| NS(encodings)[] | static array | Table of supported ENCODING pointers (Latin1, ASCII, UTF-8, UTF-16 BE/LE) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| NS(encodings)[] | const ENCODING*[7] | static | Lookup table for all supported character encodings |

## Key Functions / Methods

### NS(XmlGetUtf8InternalEncoding)
- **Signature:** `const ENCODING *NS(XmlGetUtf8InternalEncoding)()`
- **Purpose:** Return the built-in UTF-8 encoding object for internal use.
- **Inputs:** None.
- **Outputs/Return:** Pointer to internal UTF-8 ENCODING struct.
- **Side effects:** None.
- **Calls:** None visible.
- **Notes:** Entry point for obtaining standard UTF-8 encoding.

### NS(XmlGetUtf16InternalEncoding)
- **Signature:** `const ENCODING *NS(XmlGetUtf16InternalEncoding)()`
- **Purpose:** Return the built-in UTF-16 encoding, detecting system endianness at runtime.
- **Inputs:** None.
- **Outputs/Return:** Pointer to UTF-16 LE or BE ENCODING struct based on `XML_BYTE_ORDER` compile-time define or runtime endianness check.
- **Side effects:** None.
- **Calls:** None visible (relies on preprocessor).
- **Notes:** Handles three cases: explicit byte-order define (12=LE, 21=BE) or runtime detection via short pointer cast.

### NS(initScanProlog)
- **Signature:** `int NS(initScanProlog)(const ENCODING *enc, const char *ptr, const char *end, const char **nextTokPtr)`
- **Purpose:** Initialize scanner state machine for XML prolog parsing.
- **Inputs:** `enc` (current encoding), `ptr`/`end` (input buffer range), `nextTokPtr` (output).
- **Outputs/Return:** Scan result code from `initScan`.
- **Side effects:** Writes token position to `*nextTokPtr`.
- **Calls:** `initScan(NS(encodings), ..., XML_PROLOG_STATE, ...)` (defined elsewhere).
- **Notes:** Delegates to generic `initScan` with prolog state constant.

### NS(initScanContent)
- **Signature:** `int NS(initScanContent)(const ENCODING *enc, const char *ptr, const char *end, const char **nextTokPtr)`
- **Purpose:** Initialize scanner state machine for XML content parsing (post-prolog).
- **Inputs:** Same as `initScanProlog`.
- **Outputs/Return:** Scan result code.
- **Side effects:** Writes token position to `*nextTokPtr`.
- **Calls:** `initScan(NS(encodings), ..., XML_CONTENT_STATE, ...)`.
- **Notes:** Same as prolog variant but with content state constant.

### NS(XmlInitEncoding)
- **Signature:** `int NS(XmlInitEncoding)(INIT_ENCODING *p, const ENCODING **encPtr, const char *name)`
- **Purpose:** Initialize an `INIT_ENCODING` structure from an encoding name string.
- **Inputs:** `p` (init struct to fill), `encPtr` (output encoding pointer), `name` (encoding name).
- **Outputs/Return:** 1 on success, 0 if encoding name unknown.
- **Side effects:** Populates `p->initEnc.scanners[]`, `p->encPtr`, and `*encPtr`; sets `INIT_ENC_INDEX`.
- **Calls:** `getEncodingIndex(name)`, `initUpdatePosition` (defined elsewhere).
- **Notes:** Used to bootstrap encoding-specific parsing; returns 0 on `UNKNOWN_ENC` from index lookup.

### NS(findEncoding)
- **Signature:** `const ENCODING *NS(findEncoding)(const ENCODING *enc, const char *ptr, const char *end)`
- **Purpose:** Detect the character encoding declared in XML declaration text (e.g., `encoding="UTF-8"`).
- **Inputs:** `enc` (current encoding to decode declaration), `ptr`/`end` (declaration text range).
- **Outputs/Return:** Pointer to matching ENCODING struct, or NULL if not found/invalid.
- **Side effects:** Allocates local 128-byte buffer for decoded name; calls `XmlUtf8Convert`.
- **Calls:** `XmlUtf8Convert(enc, ...)`, `streqci(buf, "UTF-16")`, `getEncodingIndex(buf)`.
- **Notes:** Special case for UTF-16 (returns input `enc` if declared UTF-16 and already 2-byte encoding); otherwise looks up in `NS(encodings)` table.

### NS(XmlParseXmlDecl)
- **Signature:** `int NS(XmlParseXmlDecl)(int isGeneralTextEntity, const ENCODING *enc, const char *ptr, const char *end, const char **badPtr, const char **versionPtr, const char **encodingName, const ENCODING **encoding, int *standalone)`
- **Purpose:** Parse an XML declaration (e.g., `<?xml version="1.0" encoding="UTF-8"?>`) and extract components.
- **Inputs:** `isGeneralTextEntity` (parsing entity vs document), `enc` (current encoding), `ptr`/`end` (declaration text), output pointers.
- **Outputs/Return:** Status code from `doParseXmlDecl`.
- **Side effects:** Fills output pointers with parsed version, encoding name, encoding object, and standalone flag.
- **Calls:** `doParseXmlDecl(NS(findEncoding), ...)` (defined elsewhere).
- **Notes:** Delegates to generic parser, passing `NS(findEncoding)` as callback for encoding lookup.

## Control Flow Notes
This file operates in the **initialization and prolog-parsing phase**. Functions are called during:
1. **Startup**: `XmlGetUtf8/16InternalEncoding()` obtain base encodings; `XmlInitEncoding()` sets up scanner callbacks.
2. **Prolog scan**: `initScanProlog()` and `initScanContent()` initialize state machines.
3. **Declaration parse**: `XmlParseXmlDecl()` and `findEncoding()` detect encoding from the XML declaration header.

## External Dependencies
- **Macro wrapper:** `NS()` (likely preprocessor namespace mangling, pattern suggests C-style namespacing).
- **Functions defined elsewhere:** `initScan`, `getEncodingIndex`, `XmlUtf8Convert`, `streqci`, `doParseXmlDecl`, `initUpdatePosition`.
- **Types defined elsewhere:** `ENCODING`, `INIT_ENCODING`.
- **Constants:** `XML_BYTE_ORDER`, `XML_PROLOG_STATE`, `XML_CONTENT_STATE`, `UNKNOWN_ENC`, `ENCODING_MAX`.
