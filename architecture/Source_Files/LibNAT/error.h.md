# Source_Files/LibNAT/error.h

## File Purpose
Defines error and status codes returned by LibNAT networking library functions. Provides a centralized set of error constants (memory allocation, socket, HTTP, SSDP, UPnP failures) and an error printing utility for callers to diagnose failures.

## Core Responsibilities
- Define error/status code constants for all LibNAT subsystems
- Enable error propagation through function call stacks
- Provide human-readable error reporting via `LNat_Print_Internal_Error()`
- Distinguish error categories (generic, socket, HTTP, SSDP, UPnP)

## Key Types / Data Structures
None (header contains only preprocessor constants).

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Print_Internal_Error
- **Signature:** `void LNat_Print_Internal_Error(int error);`
- **Purpose:** Print a human-readable error message to stderr corresponding to an error code.
- **Inputs:** `int error` ΓÇö error code from this header file
- **Outputs/Return:** void (prints to stderr as side effect)
- **Side effects:** I/O to stderr
- **Calls:** Not visible in this file (implementation defined elsewhere)
- **Notes:** Maps numeric error codes to descriptive strings; called by library functions and callers to diagnose failures.

## Control Flow Notes
Not part of the main frame/update/render cycle. Used reactively whenever a library function encounters an error condition; callers check return codes and invoke `LNat_Print_Internal_Error()` to report the failure.

## External Dependencies
None. Self-contained header with no external includes; implementation of `LNat_Print_Internal_Error()` is defined elsewhere in LibNAT.
