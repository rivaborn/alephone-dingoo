# Source_Files/CSeries/csmacros.h

## File Purpose
Utility header providing common mathematical, bitwise, and memory-manipulation macros and templates for the Aleph One game engine. Enables type-safe and bounds-checked operations on objects and arrays.

## Core Responsibilities
- Arithmetic macros (MIN, MAX, FLOOR, CEILING, PIN, ABS, SGN)
- Bit manipulation (FLAG operations for 16-bit and 32-bit flags)
- Object swapping (SWAP template)
- Bounds-checked array member access with null-pointer safety
- Type-safe wrappers for `memcpy` and `memset` eliminating manual `sizeof` calls
- Rectangle dimension calculations
- Power-of-two calculation

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### SWAP
- Signature: `template <typename T> void SWAP(T& a, T& b)`
- Purpose: Exchange two objects of the same type.
- Inputs: References to two objects `a` and `b`.
- Outputs/Return: None (modifies in-place).
- Side effects: Reorders two objects.
- Calls: None.
- Notes: Requires temp variable; assumes `T` is copy-assignable.

### NextPowerOfTwo
- Signature: `static inline int NextPowerOfTwo(int n)`
- Purpose: Return the smallest power of 2 greater than or equal to `n`.
- Inputs: Integer `n`.
- Outputs/Return: Power of 2 (int).
- Side effects: None.
- Calls: None.
- Notes: Uses bit-shift loop; assumes `n >= 1`.

### GetMemberWithBounds
- Signature: `template<class T> T* GetMemberWithBounds(T* Array, const size_t Index, const size_t Number)`
- Purpose: Safely access array element with bounds checking; return null if out of bounds.
- Inputs: Array pointer, index, array size.
- Outputs/Return: Pointer to element or `NULL`.
- Side effects: None.
- Calls: None.
- Notes: Caller must check for `NULL` return; LP addition (bounds-checking accessor).

### obj_copy, objlist_copy
- Signature: `template<class T> void obj_copy(T& destination, const T& source)` and `template<class T> void objlist_copy(T* destination, const T* source, size_t num_objects)`
- Purpose: Type-safe wrapper for `memcpy` on single objects or arrays; eliminates manual `sizeof`.
- Inputs: Source and destination (pointer/reference), object count for list variant.
- Outputs/Return: None.
- Side effects: Modifies destination memory.
- Calls: `memcpy`.
- Notes: LP addition; requires objects to be trivially copyable.

### obj_set, objlist_set, obj_clear, objlist_clear
- Signature: `template<class T> void obj_set(T& object, int value)` and variants.
- Purpose: Type-safe wrappers for `memset` on single objects or arrays; `_clear` variant sets to zero.
- Inputs: Object/array pointer, value (0ΓÇô255 for `memset`), object count for list variant.
- Outputs/Return: None.
- Side effects: Modifies target memory.
- Calls: `memset` (or `obj_set` for `_clear` variants).
- Notes: LP addition; `_clear` is a common operation convenience wrapper.

## Control Flow Notes
Header-only utility; no runtime control flow. All macros and templates are compile-time constructs.

## External Dependencies
- `<string.h>` ΓÇö `memcpy`, `memset`

**Notes:**
- Macro-based bit operations are differentiated by width (16-bit vs. 32-bit); generic variants exist as well.
- Rectangle macros assume pointer-to-struct with `left`, `right`, `top`, `bottom` fields.
- All template functions guarded by `#ifdef __cplusplus` for C++ compatibility.
