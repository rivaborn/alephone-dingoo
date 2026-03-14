# Source_Files/Lua/lgc.h

## File Purpose
Header file for Lua's garbage collector implementation. Defines the tri-color marking scheme, GC states, write barrier macros for maintaining GC invariants, bit manipulation utilities, and function declarations for GC operations like incremental collection and finalization.

## Core Responsibilities
- Define garbage collector states (pause, propagate, sweep, finalize)
- Provide tri-color marking macros (white/black/gray) for incremental GC
- Implement write barrier macros to maintain GC invariants when references change
- Define bit manipulation macros for marking object metadata
- Declare GC entry points and helper functions
- Define GC threshold checking mechanism

## Key Types / Data Structures
None. (File defines only macros and function declarations; data types are in `lobject.h`.)

## Global / File-Static State
None.

## Key Functions / Methods

### luaC_step
- Signature: `void luaC_step(lua_State *L)`
- Purpose: Perform one incremental GC step
- Inputs: Lua state
- Outputs/Return: None
- Side effects: Advances GC state machine, marks/sweeps objects, updates GC memory accounting
- Calls: (defined elsewhere)
- Notes: Called by `luaC_checkGC` when GC threshold is exceeded

### luaC_fullgc
- Signature: `void luaC_fullgc(lua_State *L)`
- Purpose: Perform a complete garbage collection cycle
- Inputs: Lua state
- Outputs/Return: None
- Side effects: Collects all unreachable objects, calls finalizers
- Calls: (defined elsewhere)
- Notes: Blocking operation; transitions through all GC states

### luaC_callGCTM
- Signature: `void luaC_callGCTM(lua_State *L)`
- Purpose: Invoke garbage collection metamethods (finalizers)
- Inputs: Lua state
- Outputs/Return: None
- Side effects: Executes user-defined finalizers for userdata objects
- Calls: (defined elsewhere)

### luaC_barrierf
- Signature: `void luaC_barrierf(lua_State *L, GCObject *o, GCObject *v)`
- Purpose: Forward write barrier when a black object references a white object
- Inputs: Lua state, black object, white object reference
- Outputs/Return: None
- Side effects: Recolors target object to gray to maintain GC invariant
- Calls: (defined elsewhere)
- Notes: Prevents blackΓåÆwhite direct references in tri-color marking

### luaC_barrierback
- Signature: `void luaC_barrierback(lua_State *L, Table *t)`
- Purpose: Backward write barrier for table modifications
- Inputs: Lua state, modified table
- Outputs/Return: None
- Side effects: Adds table to GC gray list
- Calls: (defined elsewhere)
- Notes: Used when a table is modified from a black object's perspective

**Trivial helpers** (documented as macros in notes below):
- Bit manipulation: `setbits`, `resetbits`, `testbits`, `bitmask`, `l_setbit`, `resetbit`, `testbit`
- Color testing: `iswhite`, `isblack`, `isgray`, `changewhite`, `gray2black`
- Barrier triggers: `luaC_barrier`, `luaC_barriert`, `luaC_objbarrier`, `luaC_objbarriert`
- Threshold check: `luaC_checkGC`

## Control Flow Notes

**GC State Machine**: States flow through `GCSpause` ΓåÆ `GCSpropagate` ΓåÆ `GCSsweepstring` ΓåÆ `GCSsweep` ΓåÆ `GCSfinalize` ΓåÆ `GCSpause`.

**Tri-Color Marking**:
- **WHITE** (bits 0ΓÇô1): Objects not yet marked; reachable during current cycle use alternating white color (WHITE0/WHITE1) to avoid resetting marks between cycles.
- **BLACK** (bit 2): Objects fully scanned; guaranteed safe (no references to unmarked objects).
- **GRAY**: Objects discovered but not yet fully scanned; computed as `!isblack(x) && !iswhite(x)`.

**Write Barriers**: Inserted by engine code whenever a reference is created from a black object to maintain the invariant that black objects never directly point to white objects. Macros (`luaC_barrier`, `luaC_barriert`) trigger `luaC_barrierf` or `luaC_barrierback` conditionally.

**GC Trigger**: `luaC_checkGC` macro monitors heap size and calls `luaC_step` when total bytes exceed `GCthreshold`.

## External Dependencies
- `config.h` ΓÇö project configuration; checked for `HAVE_LUA`
- `lobject.h` ΓÇö defines `GCObject`, `GCheader`, `Table`, `UpVal`, `lua_State`, `lu_byte` (marked field, type tag, etc.)
- Symbols defined elsewhere: `luaD_reallocstack`, `G(L)` (global state accessor), `iscollectable`, `gcvalue`, `obj2gco`
