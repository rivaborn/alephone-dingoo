# Expat/xmltok/xmlrole.cpp

## File Purpose
Implements an XML prolog state machine that classifies tokens into semantic roles (DOCTYPE, ENTITY, NOTATION, ATTLIST, ELEMENT declarations). Processes the portion of an XML document before the root element, validating structure and identifying declaration types and their components.

## Core Responsibilities
- State machine routing via function-pointer handlers for different prolog contexts
- Token-to-role classification (returns `XML_ROLE_*` constants)
- Grammar validation for DOCTYPE, ENTITY, NOTATION, ATTLIST, and ELEMENT declarations
- Tracking nesting depth for grouped content models (parentheses in ELEMENT declarations)
- Error state management and syntax error reporting

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `PROLOG_HANDLER` | typedef | Function pointer for state handler: `int (*)(PROLOG_STATE*, int tok, const char *ptr, const char *end, ENCODING *enc)` |
| `PROLOG_STATE` | struct (external) | Contains handler function pointer and nesting level counter |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `prolog0`ΓÇô`prolog2` | static PROLOG_HANDLER | file-static | Handlers for XML declaration and prolog structure |
| `doctype0`ΓÇô`doctype5` | static PROLOG_HANDLER | file-static | Handlers for DOCTYPE declaration parsing |
| `entity0`ΓÇô`entity9` | static PROLOG_HANDLER | file-static | Handlers for ENTITY declaration parsing |
| `notation0`ΓÇô`notation4` | static PROLOG_HANDLER | file-static | Handlers for NOTATION declaration parsing |
| `attlist0`ΓÇô`attlist9` | static PROLOG_HANDLER | file-static | Handlers for ATTLIST (attribute) declaration parsing |
| `element0`ΓÇô`element7` | static PROLOG_HANDLER | file-static | Handlers for ELEMENT declaration parsing |
| `declClose`, `error`, `internalSubset` | static PROLOG_HANDLER | file-static | Special-purpose handlers |

## Key Functions / Methods

### XmlPrologStateInit
- **Signature:** `void XmlPrologStateInit(PROLOG_STATE *state)`
- **Purpose:** Initialize prolog parser state machine
- **Inputs:** Pointer to uninitialized `PROLOG_STATE` structure
- **Outputs/Return:** None (void); modifies state in-place
- **Side effects:** Sets `state->handler = prolog0` (entry point)
- **Calls:** None
- **Notes:** Must be called before any token processing

### State Handlers (e.g., prolog0, doctype0, entity2, attlist8, element7)
- **Signature:** `static int <handler_name>(PROLOG_STATE *state, int tok, const char *ptr, const char *end, const ENCODING *enc)`
- **Purpose:** Process a single token in a specific parsing state; validate grammar and emit role codes
- **Inputs:**
  - `state`: Mutable parser state (handler pointer, nesting level)
  - `tok`: Token type constant (e.g., `XML_TOK_NAME`, `XML_TOK_LITERAL`)
  - `ptr`, `end`: Pointer and end boundary of token text
  - `enc`: Encoding context (used for name matching)
- **Outputs/Return:** `XML_ROLE_*` constant indicating semantic role of token, or `XML_ROLE_ERROR` on invalid input
- **Side effects:** Updates `state->handler` to next state; increments/decrements `state->level` for nested groups
- **Calls:** `XmlNameMatchesAscii(enc, ptr, string)` to match keyword names; `syntaxError(state)` on mismatch
- **Notes:** 
  - Each handler embeds the DTD grammar via switch/case on token type
  - Transitions form a chain: `prolog0` ΓåÆ `prolog1` ΓåÆ `prolog2` or DOCTYPE/internal subset
  - `state->level` tracks parenthesis nesting in ELEMENT content models
  - No lookahead; deterministic on current token only

### syntaxError
- **Signature:** `static int syntaxError(PROLOG_STATE *state)`
- **Purpose:** Transition to error state and report syntax error
- **Inputs:** Pointer to state
- **Outputs/Return:** Always returns `XML_ROLE_ERROR`
- **Side effects:** Sets `state->handler = error`
- **Calls:** None

### error
- **Signature:** `static int error(PROLOG_STATE *state, int tok, const char *ptr, const char *end, const ENCODING *enc)`
- **Purpose:** Absorb tokens after a syntax error; do-nothing sink state
- **Inputs:** Same as state handlers
- **Outputs/Return:** Always returns `XML_ROLE_NONE`
- **Side effects:** None
- **Calls:** None
- **Notes:** Once entered, parser remains stuck here; recoverable only by external reset

## Control Flow Notes
**Prolog parsing phase** ΓÇö part of the initialization sequence before root element processing:
1. **Entry:** `XmlPrologStateInit()` sets handler to `prolog0`
2. **XML Declaration** (optional): Tokens `XML_TOK_XML_DECL` ΓåÆ role `XML_ROLE_XML_DECL`
3. **DOCTYPE** (optional): `XML_TOK_DECL_OPEN` + "DOCTYPE" ΓåÆ chain through `doctype0`ΓÇª`doctype5`
4. **Internal Subset** (optional): `XML_TOK_OPEN_BRACKET` within DOCTYPE ΓåÆ `internalSubset` handler, routes to ENTITY/NOTATION/ATTLIST/ELEMENT parsers
5. **Instance Start:** `XML_TOK_INSTANCE_START` ΓåÆ transitions to `error` handler (prolog complete, root element begins)
6. **Error Path:** Any invalid token ΓåÆ `syntaxError()` ΓåÆ `error` handler (sticky)

## External Dependencies
- `XmlNameMatchesAscii(enc, ptr, string)` ΓÇö Checks if token at `ptr` matches ASCII keyword (defined elsewhere)
- **Token constants:** `XML_TOK_*` (XML_TOK_NAME, XML_TOK_LITERAL, XML_TOK_DECL_OPEN, etc.) ΓÇö from tokenizer
- **Role constants:** `XML_ROLE_*` (XML_ROLE_DOCTYPE_NAME, XML_ROLE_ENTITY_VALUE, etc.) ΓÇö defined in xmlrole.h
- **Types:** `ENCODING` struct for character encoding info; `PROLOG_STATE` in xmlrole.h
- **Macros:** `MIN_BYTES_PER_CHAR(enc)` ΓÇö encoding-dependent byte width
