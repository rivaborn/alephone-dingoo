# Source_Files/LibNAT/utility.h

## File Purpose
Header file declaring utility functions and compile-time constants used throughout the LibNAT project. Provides macro definitions for string processing, HTTP protocol constants, and network-related size limits, plus a function declaration for string case conversion.

## Core Responsibilities
- Define macro constants for string null-termination and buffer sizing
- Define HTTP protocol constants (status codes, protocol string, default port)
- Define maximum size constraints for network strings (URLs, hostnames, resources, ports)
- Declare the `LNat_Str_To_Upper` function for ASCII uppercase conversion
- Establish common constants used across the project to avoid magic numbers

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Str_To_Upper
- **Signature:** `int LNat_Str_To_Upper(const char * str, char * dest);`
- **Purpose:** Convert a null-terminated C-string to all uppercase characters.
- **Inputs:** 
  - `str` ΓÇô input null-terminated C-string
  - `dest` ΓÇô output buffer to receive uppercase result
- **Outputs/Return:** `int` ΓÇô return status (OK on success, per comment)
- **Side effects:** Writes converted string to `dest` buffer.
- **Calls:** Not visible in this file (declaration only).
- **Notes:** Function signature only; implementation is elsewhere. Caller responsible for ensuring `dest` is large enough; no bounds checking apparent from declaration.

## Control Flow Notes
This is a header file providing declarations and constants. No control flow logic is present. Acts as an interface contract for utility functions used during initialization and throughout the application lifecycle.

## External Dependencies
- Standard C library (implementation uses standard string/character functions, not visible here)
