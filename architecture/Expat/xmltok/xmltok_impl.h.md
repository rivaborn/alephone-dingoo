# Expat/xmltok/xmltok_impl.h

## File Purpose
Defines byte-type classification constants used by the XML tokenizer to categorize characters and byte sequences. This is a foundational enum for XML character parsing, enabling efficient character-by-character analysis during tokenization.

## Core Responsibilities
- Enumerate byte type categories for XML character classification
- Distinguish between syntactically significant XML characters (delimiters, operators)
- Identify character class properties (whitespace, name starts, digits, UTF-8 leads/trails)
- Provide mnemonics for XML tokenizer logic to consume and classify input

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| (Unnamed enum) | enum | Byte type constants (BT_*) for XML character classification |

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This file does not participate in control flow directly; it is a header defining constants used by tokenizer implementations in other compilation units. The enum values classify bytes to guide parsing logic (e.g., detecting XML markup, whitespace, name tokens, UTF-8 continuations).

## External Dependencies
- `<stddef.h>` ΓÇö standard C definitions (included at end of file)

**Notes on enum constants:**
- **Markup delimiters:** `BT_LT` (`<`), `BT_GT` (`>`), `BT_SOL` (`/`), `BT_EXCL` (`!`), `BT_QUEST` (`?`), `BT_RSQB` (`]`), `BT_LSQB` (`[`)
- **Entity/name markers:** `BT_AMP` (`&`), `BT_SEMI` (`;`), `BT_NUM` (`#`), `BT_PERCNT` (`%`)
- **Operators & syntax:** `BT_EQUALS` (`=`), `BT_QUOT` (`"`), `BT_APOS` (`'`), `BT_MINUS` (`-`), `BT_COLON` (`:`)
- **Character class:** `BT_NMSTRT` (name start), `BT_NAME` (name char), `BT_DIGIT`, `BT_HEX`, `BT_S` (whitespace), `BT_CR`, `BT_LF`
- **UTF-8:** `BT_LEAD2`, `BT_LEAD3`, `BT_LEAD4` (leading bytes), `BT_TRAIL` (continuation byte)
- **Catch-all:** `BT_NONXML`, `BT_MALFORM`, `BT_OTHER`, `BT_NONASCII`
