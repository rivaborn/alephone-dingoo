# Source_Files/Lua/lapi.h

## File Purpose
Declares auxiliary functions for the Lua C API. This minimal header provides internal helper functions for Lua object manipulation at the C level, specifically for pushing tagged values onto the Lua stack.

## Core Responsibilities
- Declare internal Lua API helper functions
- Provide stack manipulation interface for TValue objects
- Support internal Lua state operations during script execution

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| TValue | struct (from lobject.h) | Tagged Lua value: unions a GCObject pointer, void pointer, number, or boolean with a type tag |

## Global / File-Static State
None.

## Key Functions / Methods

### luaA_pushobject
- Signature: `void luaA_pushobject (lua_State *L, const TValue *o);`
- Purpose: Push a Lua object (tagged value) onto the stack
- Inputs:
  - `L`: Lua state pointer
  - `o`: Pointer to a TValue (tagged Lua value)
- Outputs/Return: void
- Side effects: Modifies the Lua stack by adding an entry
- Calls: Implementation is elsewhere (not visible in this file)
- Notes: This is an auxiliary/internal function; the actual implementation must be in a `.c` file to avoid multiple definitions.

## Control Flow Notes
This header does not directly participate in init/frame/update/render cycles. It provides low-level stack manipulation utilities called by other Lua API functions during script execution and CΓÇôLua interop.

## External Dependencies
- `config.h`: Conditional compilation guard (`HAVE_LUA`)
- `lobject.h`: Provides `TValue` definition and Lua object type system (tags, GCObject union, type checking macros)
- Implicit: `lua.h` (referenced in copyright notice; provides `lua_State`)
