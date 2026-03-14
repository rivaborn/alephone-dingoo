# Expat/xmltok/iasciitab.h

## File Purpose

A lookup table that classifies ASCII byte values (0x00ΓÇô0x7F) into character type categories for XML tokenization. This variant treats carriage return (0xD) as whitespace (`BT_S`) rather than a distinct token type, diverging from the standard `asciitab.h`.

## Core Responsibilities

- Provides 256-entry character classification constants for the XML parser's tokenizer
- Maps each byte value to a `BT_*` token type (e.g., `BT_NMSTRT`, `BT_DIGIT`, `BT_S`, `BT_NONXML`)
- Distinguishes XML-valid name start characters, name continuation characters, digits, hex digits, and whitespace
- Handles special characters (delimiters: `<`, `>`, `=`, `/`, `&`, `%`, etc.)
- Designed to be included into tokenizer implementations via preprocessor directives

## Key Types / Data Structures

None.

## Global / File-Static State

None explicitly named in this file. The table data (implicit array of `BT_*` constants indexed by byte value) is the sole content.

## Key Functions / Methods

None.

## Control Flow Notes

This file is **not executable code**ΓÇöit is a preprocessor-included data table. The actual array declaration wraps this content in an includer file (e.g., `iasciitab.c` or a similar `.c` file that declares the array). During XML tokenization, the parser indexes this table by the current byte value to determine how to classify and handle that character in the parsing state machine.

## External Dependencies

- **BT_* constants**: defined elsewhere (likely in `xmltok.h` or similar header); represents token/byte types:
  - `BT_NONXML`, `BT_S` (whitespace), `BT_LF` (linefeed), `BT_NAME`, `BT_NMSTRT` (name start), `BT_DIGIT`, `BT_HEX`
  - Special characters: `BT_LT`, `BT_GT`, `BT_EQUALS`, `BT_AMP`, `BT_PERCNT`, `BT_SEMI`, `BT_COLON`, etc.
- Included by tokenizer implementations (not a standalone header)
