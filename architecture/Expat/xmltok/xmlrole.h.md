# Expat/xmltok/xmlrole.h

## File Purpose
Defines XML structural role enumerations and prolog state management for the Expat tokenizer. Provides a state machine interface for categorizing tokens during XML prologue parsing and determining their semantic role (e.g., attribute name, element declaration, entity definition).

## Core Responsibilities
- Define enum constants for XML structural roles (XML_ROLE_* values) used during prologue parsing
- Define the PROLOG_STATE structure for tracking parser state during prologue processing
- Initialize and manage prologue parsing state
- Provide macro interface for invoking token role handlers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| PROLOG_STATE | struct | Holds parsing state during XML prologue: handler function pointer and nesting level |

## Global / File-Static State
None.

## Key Functions / Methods

### XmlPrologStateInit
- **Signature:** `void XMLTOKAPI XmlPrologStateInit(PROLOG_STATE *)`
- **Purpose:** Initialize a PROLOG_STATE structure to begin prologue parsing
- **Inputs:** Pointer to uninitialized PROLOG_STATE
- **Outputs/Return:** None (initializes via pointer)
- **Side effects:** Modifies PROLOG_STATE fields (handler, level)
- **Calls:** Defined elsewhere (implementation in xmlrole.c)
- **Notes:** Typically called at document parse start to prepare for prologue scanning

### XmlTokenRole (macro)
- **Signature:** `XmlTokenRole(state, tok, ptr, end, enc)` ΓåÆ evaluates to `(state)->handler(state, tok, ptr, end, enc)`
- **Purpose:** Dispatch a token to the appropriate role handler based on current prologue state
- **Inputs:** PROLOG_STATE*, token type (from xmltok.h), buffer range (ptrΓÇôend), ENCODING*
- **Outputs/Return:** int (XML_ROLE_* constant indicating semantic role)
- **Side effects:** Handler may advance state (via level field) or modify state
- **Calls:** User-supplied handler function stored in PROLOG_STATE.handler
- **Notes:** Handler pattern allows state-machine-driven role assignment; actual handler implementation varies

## Control Flow Notes
This file is part of the **prologue parsing phase** (XML declaration, DOCTYPE, element/attribute declarations, entity definitions). The PROLOG_STATE and its handler implement a state machine that transitions through prologue rules and classifies tokens by their semantic role. After prologue completion, the parser switches to XML_CONTENT_STATE (defined in xmltok.h).

## External Dependencies
- **Includes:** `xmltok.h` (defines ENCODING struct, XML_TOK_* token constants, related macros)
- **Defined elsewhere:** ENCODING struct and token type constants used as inputs to role handlers
