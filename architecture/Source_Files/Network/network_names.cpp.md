# Source_Files/Network/network_names.cpp

## File Purpose
Manages registration and unregistration of AppleTalk network entity names for the local game instance. Provides a wrapper around the Mac Toolbox NBP (Name Binding Protocol) to advertise the game's presence on the network with a socket number and type identifier.

## Core Responsibilities
- Register a single entity name on the network via AppleTalk NBP
- Store the names table entry in system heap to survive application crashes
- Construct adjusted type names by appending version numbers
- Unregister and deallocate the names table entry on shutdown
- Optionally support modem-based name registration (test mode)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NamesTableEntryPtr` | typedef (pointer) | Pointer to AppleTalk names table entry |
| `MPPParamBlock` | struct (from AppleTalk.h) | Parameter block for NBP operations |
| `Str255` | typedef (Pascal string) | Mac OS fixed-length string type |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `myNTEName` | `NamesTableEntryPtr` | static | Holds the registered names table entry; persists for the lifetime of registration |

## Key Functions / Methods

### NetRegisterName
- **Signature:** `OSErr NetRegisterName(unsigned char *name, unsigned char *type, short version, short socketNumber)`
- **Purpose:** Allocate and register an entity name on the network for the given socket.
- **Inputs:**
  - `name` (Pascal string, nullable) ΓÇô entity name; if NULL, replaced with system user name
  - `type` (Pascal string) ΓÇô entity type
  - `version` (short) ΓÇô appended to type for uniqueness
  - `socketNumber` (short) ΓÇô DDP socket to bind the name to
- **Outputs/Return:** `OSErr` ΓÇô noErr on success, error code on failure
- **Side effects:**
  - Allocates `myNTEName` in system heap via `NewPtrSysClear()`
  - Allocates temporary `MPPParamBlock` on stack
  - Registers name via `PRegisterName()` with verify and retry parameters
- **Calls:** `pstrcpy()`, `GetString()`, `psprintf()`, `NBPSetNTE()`, `PRegisterName()`, `MemError()`, `NewPtrSysClear()`, `DisposePtr()`
- **Notes:**
  - Asserts that `myNTEName` is NULL before allocation (enforces single registration at a time)
  - Uses system heap allocation to prevent pointer corruption if app crashes while name is registered
  - Retry every 16 ticks, 4 attempts total (64 ticks max wait)
  - Supports optional `MODEM_TEST` mode that delegates to `ModemRegisterName()`

### NetUnRegisterName
- **Signature:** `OSErr NetUnRegisterName(void)`
- **Purpose:** Deallocate and unregister the entity name previously registered by `NetRegisterName()`.
- **Inputs:** None
- **Outputs/Return:** `OSErr` ΓÇô noErr on success or if no name was registered
- **Side effects:**
  - Deallocates `myNTEName` via `DisposePtr()` and sets it to NULL
  - Allocates temporary `MPPParamBlock` on stack
  - Calls `PRemoveName()` to unregister from AppleTalk
- **Calls:** `MemError()`, `NewPtrClear()`, `PRemoveName()`, `DisposePtr()`
- **Notes:**
  - Safe to call multiple times (checks if `myNTEName` is non-NULL)
  - Extracts `entityPtr` from the names table entry's internal structure before removal
  - Supports optional `MODEM_TEST` mode that delegates to `ModemUnRegisterName()`

## Control Flow Notes
This module participates in **initialization** (register name at startup) and **shutdown** (unregister name at exit). The lifecycle is external to this file; callers must invoke `NetRegisterName()` once during game startup and `NetUnRegisterName()` during shutdown. Only one name can be registered at a time (enforced by assertion on `myNTEName`).

## External Dependencies
- **`#include "macintosh_cseries.h"`** ΓÇô Mac platform utilities (string functions, memory allocation)
- **`#include "macintosh_network.h"`** ΓÇô Game network header (provides function prototypes and AppleTalk type definitions)
- **`#include <AppleTalk.h>`** (via macintosh_network.h) ΓÇô Mac Toolbox AppleTalk definitions (NBP, NamesTableEntry, etc.)
- **Defined elsewhere:** `GetString()`, `pstrcpy()`, `psprintf()`, `NewPtrClear()`, `NewPtrSysClear()`, `DisposePtr()`, `MemError()`, `NBPSetNTE()`, `PRegisterName()`, `PRemoveName()`, `ModemRegisterName()`, `ModemUnRegisterName()`
