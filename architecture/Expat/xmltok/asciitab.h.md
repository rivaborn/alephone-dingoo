# Expat/xmltok/asciitab.h

## File Purpose
Defines a static lookup table mapping ASCII byte values (0x00ΓÇô0x7F) to XML character type classifications. This table enables fast, single-lookup character categorization during XML tokenization without repeated conditional branching.

## Core Responsibilities
- Classifies ASCII bytes into XML-relevant categories (whitespace, digits, name characters, special symbols)
- Provides O(1) character type lookup by byte value as table index
- Supports XML tokenizer decisions (e.g., valid name-start characters, operators, delimiters)
- Handles ASCII control characters, spaces, line feeds, and carriage returns
- Distinguishes hex digits, name characters, and name-start characters for identifier parsing

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Implicit lookup table | `BT_*` constants (array) | File-level (header) | 256-entry byte classification table indexed by ASCII value 0x00ΓÇô0x7F (conceptually; actual array defined in consuming `.c` file) |

## Key Functions / Methods
None.

## Control Flow Notes
This file is a data definition header, not executable code. It is `#include`d by XML tokenizer implementation files (likely `xmltok.c` or similar) to define or initialize a byte-type lookup table used during tokenization phases. The table classifies every ASCII byte into one of ~20 type categories (e.g., `BT_S`, `BT_DIGIT`, `BT_NMSTRT`), enabling fast character validation without repeated conditionals.

## External Dependencies
- **Character type constants**: References undeclared `BT_*` macro/enum constants (e.g., `BT_NONXML`, `BT_S`, `BT_DIGIT`, `BT_NMSTRT`, `BT_HEX`, `BT_NAME`, `BT_EXCL`) ΓÇö defined elsewhere, likely in a shared header such as `xmltok.h` or an internal type definition.
- **License**: Mozilla Public License / GNU GPL dual-licensed.
