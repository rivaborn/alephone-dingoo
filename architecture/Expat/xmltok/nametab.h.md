# Expat/xmltok/nametab.h

## File Purpose
Static lookup table header for XML name character validation in the Expat parser. Provides bitmap and page-mapping tables to efficiently classify Unicode characters as valid XML name start/continuation characters.

## Core Responsibilities
- Define `namingBitmap[]`: Bit patterns encoding valid characters for XML identifiers across Unicode ranges
- Define `nmstrtPages[]`: Page indices mapping first-byte code points to bitmap sections for name-start characters
- Define `namePages[]`: Page indices mapping first-byte code points to bitmap sections for name-continuation characters
- Support rapid character classification without requiring full Unicode property lookups

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `namingBitmap` | `unsigned[]` (128 elements) | static | Bit patterns indicating valid XML name characters; each bit represents one code point within a 128-bit window |
| `nmstrtPages` | `unsigned char[]` (256 elements) | static | Page index lookup for name-start character validation; maps high byte to bitmap page index |
| `namePages` | `unsigned char[]` (256 elements) | static | Page index lookup for name-continuation character validation; maps high byte to bitmap page index |

## Key Functions / Methods
None.

## Control Flow Notes
These tables are passive lookup data consumed at runtime by tokenizer functions (likely in other `.c` files). Used during XML parsing to validate element/attribute names character-by-character via:
1. Extract high byte of Unicode code point ΓåÆ index into `nmstrtPages` or `namePages`
2. Retrieve page index
3. Use page index and low bits to address into `namingBitmap`
4. Check specific bit to determine validity

## External Dependencies
- Defined elsewhere: consuming code that performs character lookups (tokenizer/validator in sibling `.c` files)
- No includes visible; pure data-only header

## Notes
- Tables are generated, not hand-written (typical for Unicode property lookups)
- Supports both name-start (first character) and name-continuation (subsequent characters) rules per XML spec
- Bit-level packing optimizes memory footprint for large character range coverage
