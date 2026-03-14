# Source_Files/Misc/VecOps.h

## File Purpose
Header file providing template-based 3D vector operations for the Aleph One game engine. Defines low-level inline functions for common mathematical operations on vectors represented as 3-element arrays, with support for type conversion between different numeric types.

## Core Responsibilities
- Copy vectors between different types
- Vector arithmetic (addition, subtraction, in-place modification)
- Scalar multiplication (per-component and in-place)
- Dot product (scalar product) calculation
- Cross product (vector product) calculation
- Enable type-safe conversions during vector operations via templating

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### VecCopy
- **Signature:** `template <class T0, class T1> inline void VecCopy(const T0* V0, T1* V1)`
- **Purpose:** Copy source vector to destination, with optional type conversion per component.
- **Inputs:** V0 (const pointer to 3-element source array), V1 (pointer to 3-element destination array)
- **Outputs/Return:** None; modifies V1 in-place.
- **Side effects:** Writes to V1[0..2].
- **Calls:** None.
- **Notes:** Supports heterogeneous types; each component cast via `T1(V0[i])`.

### VecAdd
- **Signature:** `template <class T0, class T1, class T2> inline void VecAdd(const T0* V0, const T1* V1, T2* V2)`
- **Purpose:** Add two vectors component-wise and store result.
- **Inputs:** V0, V1 (const pointers to source arrays); V2 (pointer to result array)
- **Outputs/Return:** None; writes to V2.
- **Side effects:** Modifies V2[0..2].
- **Calls:** None.
- **Notes:** Supports three independent types; result type T2 is inferred at call site.

### VecSub
- **Signature:** `template <class T0, class T1, class T2> inline void VecSub(const T0* V0, const T1* V1, T2* V2)`
- **Purpose:** Subtract V1 from V0 and store result.
- **Inputs:** V0, V1 (const pointers); V2 (pointer to result)
- **Outputs/Return:** None; writes to V2.
- **Side effects:** Modifies V2[0..2].
- **Calls:** None.

### VecAddTo / VecSubFrom
- **Signature:** `template <class T0, class T1> inline void VecAddTo(T0* V0, const T1* V1)` (and VecSubFrom variant)
- **Purpose:** In-place vector addition/subtraction (V0 += V1 or V0 -= V1).
- **Inputs:** V0 (mutable pointer), V1 (const pointer)
- **Outputs/Return:** None; modifies V0 in-place.
- **Side effects:** Modifies V0[0..2].
- **Calls:** None.

### VecScalarMult / VecScalarMultTo
- **Signature:** `template <class T0, class TS, class T1> inline void VecScalarMult(const T0* V0, const TS& S, T1* V1)` (and in-place variant)
- **Purpose:** Multiply vector by scalar; create new result or modify in-place.
- **Inputs:** V0 (const vector), S (scalar reference), V1 (result pointer in non-mutating version)
- **Outputs/Return:** None; writes to V1 or V.
- **Side effects:** Modifies output array or V[0..2].
- **Calls:** None.
- **Notes:** Scalar multiplied first, then cast to result type.

### ScalarProd
- **Signature:** `template <class T> inline T ScalarProd(const T* V0, const T* V1)`
- **Purpose:** Compute dot product (scalar product) of two vectors.
- **Inputs:** V0, V1 (const pointers to same type)
- **Outputs/Return:** T (dot product value).
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Single return expression; requires both vectors same type T.

### VectorProd
- **Signature:** `template <class T0, class T1, class T2> inline void VectorProd(const T0* V0, const T1* V1, T2* V2)`
- **Purpose:** Compute cross product V0 ├ù V1 and store result.
- **Inputs:** V0, V1 (const pointers); V2 (pointer to result)
- **Outputs/Return:** None; writes to V2.
- **Side effects:** Modifies V2[0..2].
- **Calls:** None.
- **Notes:** Standard cross-product formula; result type may differ from inputs.

## Control Flow Notes
Utility header with no control-flow integration. Included by other files requiring vector math (typically physics, geometry, or rendering systems). No frame-time or shutdown dependencies.

## External Dependencies
None. Self-contained header using only C++ template syntax.
