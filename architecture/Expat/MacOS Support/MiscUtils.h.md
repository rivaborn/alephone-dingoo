# Expat/MacOS Support/MiscUtils.h

## File Purpose
A macOS-specific utility header providing inline template functions and helper utilities for mathematical operations, modular arithmetic, and Pascal-to-C string conversions. Intended as a collection of convenience functions for the broader Expat codebase.

## Core Responsibilities
- Provide generic template utilities (squaring function)
- Round floating-point values to integers
- Perform positive-range modular arithmetic with wraparound
- Convert between Pascal and C string formats (legacy macOS string types)
- Document available Standard Template Library functions for reference

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Str255 | typedef (from MacTypes.h) | Fixed-length Pascal string type (256 chars max) |

## Global / File-Static State
None.

## Key Functions / Methods

### sqr
- **Signature:** `template<class T> inline T sqr(const T& x)`
- **Purpose:** Compute the square of a value.
- **Inputs:** Generic type `T` value `x`
- **Outputs/Return:** `T` (same type as input, squared)
- **Side effects:** None
- **Calls:** None (pure computation)
- **Notes:** Template allows use with int, float, double, etc.

### irint
- **Signature:** `inline int irint(double x)`
- **Purpose:** Round a double-precision float to the nearest integer (mimics Sun math library behavior).
- **Inputs:** `double x`
- **Outputs/Return:** `int` (rounded value)
- **Side effects:** None
- **Calls:** None
- **Notes:** Handles negative values correctly by negating the rounding operation.

### pos_mod
- **Signature:** `inline int pos_mod(int i, int n)`
- **Purpose:** Compute modulo with guaranteed positive result (wraps negative remainders).
- **Inputs:** `int i` (value), `int n` (modulus base)
- **Outputs/Return:** `int` (non-negative remainder)
- **Side effects:** None
- **Calls:** None
- **Notes:** Ensures result is always in range [0, n-1] even for negative `i`.

### mod_incr / mod_decr
- **Signature:** `inline int mod_incr(int i, int n)` and `inline int mod_decr(int i, int n)`
- **Purpose:** Increment or decrement an index with wraparound in modular space.
- **Inputs:** `int i` (current index), `int n` (modulus size)
- **Outputs/Return:** `int` (wrapped next/previous index)
- **Side effects:** None
- **Calls:** None
- **Notes:** Optimized for circular buffer or ring buffer navigation.

### Pas2C
- **Signature:** `inline void Pas2C(Str255 InStr, Str255 OutStr)`
- **Purpose:** Convert Pascal string (length-prefixed) to C string (null-terminated).
- **Inputs:** `Str255 InStr` (Pascal string; InStr[0] = length)
- **Outputs/Return:** `void` (result written to `OutStr`)
- **Side effects:** Modifies `OutStr`
- **Calls:** None
- **Notes:** Reads length from first byte, copies bytes 1..len to output, null-terminates.

### C2Pas
- **Signature:** `inline void C2Pas(Str255 InStr, Str255 OutStr, int MaxLen=255)`
- **Purpose:** Convert C string (null-terminated) to Pascal string (length-prefixed).
- **Inputs:** `Str255 InStr` (C string), `int MaxLen` (max output length, default 255)
- **Outputs/Return:** `void` (result written to `OutStr`)
- **Side effects:** Modifies `OutStr`
- **Calls:** None
- **Notes:** Scans for null terminator up to MaxLen, stores length in first byte, reverses copy loop to handle in-place conversion.

## Control Flow Notes
These are pure utility functions with no control flow dependencies. Likely called throughout the Expat codebase as needed for math, string handling, or circular buffer operations. No init/frame/shutdown participation inferred.

## External Dependencies
- `#include <algorithm.h>` ΓÇô Standard Template Library (note: legacy include syntax)
- `#include <MacTypes.h>` ΓÇô macOS/Classic Mac Toolbox types (Str255 definition)
- `using namespace std;` ΓÇô Standard namespace in scope
