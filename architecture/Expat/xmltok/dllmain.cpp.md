# Expat/xmltok/dllmain.cpp

## File Purpose
Windows DLL (Dynamic Link Library) entry point for the Expat XML tokenizer library. Provides the required `DllMain` function that Windows calls when the DLL is loaded into or unloaded from a process. This is a minimal stub implementation with no custom initialization logic.

## Core Responsibilities
- Export the mandatory `DllMain` entry point required by Windows for DLL execution
- Signal successful DLL attachment/detachment to the operating system
- Comply with Windows DLL module initialization conventions

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### DllMain
- **Signature:** `BOOL WINAPI DllMain(HANDLE hInst, ULONG ul_reason_for_call, LPVOID lpReserved)`
- **Purpose:** Windows DLL entry point called during DLL load/unload and thread attach/detach events.
- **Inputs:**
  - `hInst`: Handle to the DLL module instance
  - `ul_reason_for_call`: Reason for the call (e.g., `DLL_PROCESS_ATTACH`, `DLL_PROCESS_DETACH`)
  - `lpReserved`: Reserved; unused
- **Outputs/Return:** `TRUE` (success); unconditionally returns `TRUE` regardless of reason
- **Side effects:** None; minimal stub implementation
- **Calls:** None
- **Notes:** No differentiation by reason code; all scenarios (process attach, process detach, thread attach, thread detach) are handled identically with immediate success

## Control Flow Notes
Called by Windows loader at DLL attachment (process or thread load) and detachment. Not inferable as part of game engine initializationΓÇöthis is pure OS-level module management.

## External Dependencies
- `<windows.h>` ΓÇö Windows API header (HANDLE, BOOL, WINAPI, LPVOID types)
- Expat library context (inferred from file path)

---

**Note:** This file is from the Expat XML parser library, not a game engine. It is a standard Windows DLL stub with no game-specific or engine-specific logic.
