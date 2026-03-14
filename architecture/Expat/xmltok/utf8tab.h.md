# Expat/xmltok/utf8tab.h

## File Purpose
A lookup table for UTF-8 byte classification used by Expat's XML tokenizer. Maps each possible byte value (0x00ΓÇô0xFF) to a byte type constant indicating whether it is a UTF-8 lead byte, continuation byte, invalid, or non-XML. This table enables efficient single-byte-lookup validation and parsing of UTF-8 encoded XML.

## Core Responsibilities
- Define or include a 256-entry byte classification lookup table
- Classify bytes as UTF-8 lead bytes (2, 3, or 4-byte sequences), continuation bytes, malformed bytes, or non-XML characters
- Enable O(1) UTF-8 validation during XML tokenization
- Support correct multi-byte UTF-8 character boundary detection

## Key Types / Data Structures
None (this is a data table, not type definitions).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| (implicit table) | array of byte type constants | global/static | 256-entry lookup table indexed by byte value (0x00ΓÇô0xFF); each entry holds a `BT_*` constant classifying that byte |

## Key Functions / Methods
None (this is a data file; no executable functions).

## Control Flow Notes
This table is referenced during XML tokenization (frame/update phase) when parsing UTF-8 encoded input. The tokenizer likely indexes this table on each byte to determine whether a byte is a UTF-8 lead byte, continuation byte, or invalid, guiding multi-byte sequence parsing and validation.

## External Dependencies
- `BT_TRAIL`, `BT_LEAD2`, `BT_LEAD3`, `BT_LEAD4`, `BT_NONXML`, `BT_MALFORM` ΓÇö byte type constants (defined elsewhere, likely in `xmltok.h` or similar)
- Expat XML tokenizer ΓÇö consumes this table for UTF-8 byte classification

## Notes
- The visible portion shows bytes 0x80ΓÇô0xFF; bytes 0x00ΓÇô0x7F (ASCII) are likely defined elsewhere (not shown).
- UTF-8 encoding rules are encoded via this table:
  - 0x80ΓÇô0xBF: continuation bytes (`BT_TRAIL`)
  - 0xC0ΓÇô0xDF: 2-byte lead (`BT_LEAD2`)
  - 0xE0ΓÇô0xEF: 3-byte lead (`BT_LEAD3`)
  - 0xF0ΓÇô0xF3: 4-byte lead (`BT_LEAD4`)
  - 0xF4ΓÇô0xFF: invalid or non-XML (`BT_NONXML`, `BT_MALFORM`)
- This is a generated or manually optimized table for performance-critical byte classification during parsing.
