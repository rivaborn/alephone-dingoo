# Source_Files/Lua/llex.h

## File Purpose
Header for Lua's lexical analyzer. Defines token types, semantic information structures, lexer state, and the public API for tokenization of Lua source code.

## Core Responsibilities
- Define reserved word tokens and operator token constants (enum RESERVED)
- Define Token and SemInfo structures for representing parsed tokens and their values
- Define LexState structure maintaining lexer state during tokenization
- Declare lexer initialization, input setup, and token manipulation functions
- Provide error reporting utilities for lexical/syntax errors

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SemInfo | union | Holds token semantic value: either `lua_Number` (for numeric literals) or `TString*` (for identifiers/strings) |
| Token | struct | Token representation: `token` type (int) and `seminfo` (SemInfo) |
| LexState | struct | Lexer state: current character, line tracking, current/lookahead tokens, input stream (ZIO), buffer, source name, decimal point locale |
| RESERVED | enum | Token type constants starting at FIRST_RESERVED (257): reserved words (TK_AND, TK_IF, etc.) and operators (TK_CONCAT, TK_EQ, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| luaX_tokens[] | const char*[] | global data | Token name strings; indexed by token type for error messages |

## Key Functions / Methods

### luaX_init
- Signature: `void luaX_init(lua_State *L)`
- Purpose: Initialize lexer subsystem at startup
- Inputs: Lua state
- Outputs/Return: void
- Side effects: Sets up global lexer data structures
- Notes: Called once during Lua initialization

### luaX_setinput
- Signature: `void luaX_setinput(lua_State *L, LexState *ls, ZIO *z, TString *source)`
- Purpose: Prepare lexer to tokenize from a new input stream
- Inputs: Lua state, lexer state, input stream, source name
- Outputs/Return: void
- Side effects: Initializes LexState fields; resets current token and lookahead
- Notes: Called before tokenizing each new source file/chunk

### luaX_next
- Signature: `void luaX_next(LexState *ls)`
- Purpose: Advance to the next token in input
- Inputs: Lexer state
- Outputs/Return: void
- Side effects: Shifts lookahead token to current (`ls->t`), scans new lookahead
- Notes: Core tokenization step; main parsing loop calls this repeatedly

### luaX_lookahead
- Signature: `void luaX_lookahead(LexState *ls)`
- Purpose: Scan next token without advancing current token
- Inputs: Lexer state
- Outputs/Return: void
- Side effects: Updates `ls->lookahead` field
- Notes: Enables LL(1) predictive parsing in the parser

## Notes
- `luaX_newstring()` ΓÇö Creates or interns a Lua string token from a buffer
- `luaX_lexerror()`, `luaX_syntaxerror()` ΓÇö Report errors with line number and token context
- `luaX_token2str()` ΓÇö Converts token type to human-readable name (debugging/errors)

## Control Flow Notes
**Initialization phase:** `luaX_init()` once at startup; `luaX_setinput()` per source.  
**Parsing phase:** Parser repeatedly calls `luaX_next()` to consume tokens; calls `luaX_lookahead()` before `luaX_next()` to enable LL(1) predictive decisions (parser checks `ls->lookahead.token` to decide reductions). Tokenization rules (digits, operators, keywords) implemented in llex.c.

## External Dependencies
- `config.h` ΓÇö Compilation config; guards module with `HAVE_LUA`
- `lobject.h` ΓÇö Type definitions: `TString`, `lua_Number`
- `lzio.h` ΓÇö Buffered input: `ZIO` (stream), `Mbuffer` (token buffer)
- `lua_State` ΓÇö Lua runtime (opaque; defined elsewhere)
- `FuncState` ΓÇö Parser state (opaque to lexer; stored as pointer in LexState)
- Macros `LUAI_FUNC`, `LUAI_DATA` ΓÇö Function/data visibility annotations
