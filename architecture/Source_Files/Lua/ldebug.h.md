# Source_Files/Lua/ldebug.h

## File Purpose
Header for Lua's Debug Interface auxiliary functions module. Provides macro utilities and declares error reporting and code validation functions used throughout the Lua runtime to handle errors and debug hooks.

## Core Responsibilities
- Define debugging macros for bytecode operations (program counter relative offset, line info lookup, hook count reset)
- Declare type-specific error reporting functions (type errors, concatenation, arithmetic, ordering)
- Declare general runtime error and error message dispatch functions
- Declare code and instruction validation functions for bytecode integrity

## Key Types / Data Structures
None defined in this file. (Uses `lua_State`, `TValue`, `StkId`, `Proto` from `lstate.h`.)

## Global / File-Static State
None.

## Key Functions / Methods

### luaG_typeerror
- Signature: `void luaG_typeerror(lua_State *L, const TValue *o, const char *opname)`
- Purpose: Report a type error when an operation receives an invalid type.
- Inputs: `L` (Lua state), `o` (value with wrong type), `opname` (name of operation that failed)
- Outputs/Return: None (signals error to Lua state)
- Side effects: Modifies error state in `lua_State`
- Calls: (defined elsewhere in ldebug.c)

### luaG_concaterror
- Signature: `void luaG_concaterror(lua_State *L, StkId p1, StkId p2)`
- Purpose: Report a concatenation error when operands cannot be concatenated.
- Inputs: `L` (Lua state), `p1`, `p2` (stack indices of operands)
- Outputs/Return: None
- Side effects: Modifies error state in `lua_State`

### luaG_aritherror
- Signature: `void luaG_aritherror(lua_State *L, const TValue *p1, const TValue *p2)`
- Purpose: Report an arithmetic operation error (invalid operand types for +, ΓêÆ, ├ù, etc.).
- Inputs: `L` (Lua state), `p1`, `p2` (operands that failed)
- Outputs/Return: None
- Side effects: Modifies error state in `lua_State`

### luaG_ordererror
- Signature: `int luaG_ordererror(lua_State *L, const TValue *p1, const TValue *p2)`
- Purpose: Report an order comparison error (invalid operand types for <, >, Γëñ, ΓëÑ).
- Inputs: `L` (Lua state), `p1`, `p2` (values being compared)
- Outputs/Return: Integer (error code or status)
- Side effects: Modifies error state

### luaG_runerror
- Signature: `void luaG_runerror(lua_State *L, const char *fmt, ...)`
- Purpose: Report a formatted runtime error (variadic printf-style error reporting).
- Inputs: `L` (Lua state), `fmt` (format string), variadic arguments
- Outputs/Return: None
- Side effects: Modifies error state; generates formatted message

### luaG_errormsg
- Signature: `void luaG_errormsg(lua_State *L)`
- Purpose: Dispatch and propagate an error message that has been set in the Lua state.
- Inputs: `L` (Lua state with pending error)
- Outputs/Return: None
- Side effects: Triggers error handling (longjmp or error handler invocation)

### luaG_checkcode
- Signature: `int luaG_checkcode(const Proto *pt)`
- Purpose: Validate bytecode integrity of a function prototype.
- Inputs: `pt` (function prototype with bytecode to validate)
- Outputs/Return: Integer status (0 if valid, non-zero if invalid)
- Side effects: None (read-only validation)

### luaG_checkopenop
- Signature: `int luaG_checkopenop(Instruction i)`
- Purpose: Validate that an instruction is a valid "open" (variadic-argument-related) opcode.
- Inputs: `i` (bytecode instruction)
- Outputs/Return: Integer status
- Side effects: None

## Macros
- **pcRel(pc, p)**: Compute relative bytecode offset from absolute pointer; subtracts base and adjusts by ΓêÆ1.
- **getline(f, pc)**: Retrieve source line number for a bytecode instruction; returns 0 if lineinfo unavailable.
- **resethookcount(L)**: Reset hook invocation counter to baseline on the Lua state.

## Control Flow Notes
This module is invoked reactively during VM execution: error-reporting functions are called when runtime errors occur (type mismatches, invalid operations, etc.). The `luaG_errormsg` function acts as the error dispatch point, routing errors to registered error handlers or terminating execution. Validation functions (`luaG_checkcode`, `luaG_checkopenop`) run during bytecode loading/verification.

## External Dependencies
- **Includes**: `config.h`, `lstate.h`
- **Types used** (defined elsewhere): `lua_State`, `TValue`, `StkId`, `Proto`, `Instruction`
- **LUAI_FUNC macro**: Marks these as Lua internal C API functions
