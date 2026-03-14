# Expat/MacOS Support/SimpleVec.h

## File Purpose
Defines custom template-based vector and array containers (`simple_vector` and `simple_array`) as alternatives to STL containers. These containers offer simple resizable storage with STL-like interfaces, designed to work around allocator issues encountered with standard STL containers on some platforms.

## Core Responsibilities
- Provide heap-allocated resizable vector storage via `simple_vector<T>`
- Provide fixed-dimension array-of-vectors storage via `simple_array<T,N>`
- Manage memory allocation and deallocation with explicit `allocate()` / `deallocate()` calls
- Expose STL-compatible interfaces: indexing, iterator-like accessors, assignment, swapping
- Support initialization from raw pointers, copy construction, and range-based construction

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `simple_vector<T>` | template class | Resizable array of type T with explicit allocation |
| `simple_array<T,N>` | template class | Fixed-dimension array; stores N├ùlength elements linearly but accessed as row vectors |

## Global / File-Static State
None

## Key Functions / Methods

### simple_vector::allocate (private)
- **Purpose:** Allocate heap memory for the vector's contents.
- **Side effects:** Allocates `new T[length]` if length > 0; sets contents to null and length to 0 otherwise.
- **Notes:** Called by constructors and `reallocate()`.

### simple_vector::deallocate (private)
- **Purpose:** Release heap memory.
- **Side effects:** Calls `delete []contents` if length > 0.

### simple_vector::copy_in (private)
- **Purpose:** Copy elements from an external array into contents.
- **Inputs:** `T *newctnts` ΓÇô source array of at least length elements.
- **Side effects:** Overwrites existing contents element-by-element.

### simple_vector constructor (default)
- **Signature:** `simple_vector(int _length = 0)`
- **Purpose:** Construct vector with given length (default 0).
- **Outputs/Return:** Initialized vector with allocated memory.

### simple_vector copy constructor
- **Signature:** `simple_vector(simple_vector<T> &v)`
- **Purpose:** Deep copy another vector.
- **Side effects:** Allocates new memory and copies all elements.

### simple_vector constructor (from pointer range)
- **Signature:** `simple_vector(T *v_begin, T *v_end)`
- **Purpose:** Construct from a pointer range.
- **Calls:** `copy()` (from `<algorithm.h>`).

### simple_vector::operator[]
- **Signature:** `T &operator[](int indx)`
- **Purpose:** Provide index-based element access.
- **Outputs/Return:** Reference to element at index.
- **Notes:** No bounds checking.

### simple_vector::operator+
- **Signature:** `T *operator+(int indx)`
- **Purpose:** Pointer arithmetic; return address of element at offset.
- **Outputs/Return:** `contents + indx` (pointer to element).

### simple_vector::reallocate
- **Signature:** `void reallocate(int _length)`
- **Purpose:** Resize vector, destroying previous contents.
- **Side effects:** Deallocates old memory, updates length, allocates new memory.

### simple_vector::operator=
- **Signature:** `simple_vector &operator=(simple_vector<T> &v)`
- **Purpose:** Assignment operator; copy another vector's contents.
- **Side effects:** Calls `reallocate()` then `copy_in()`.

### simple_vector iterator accessors
- **Methods:** `begin()`, `end()`, `rbegin()`, `rend()`
- **Purpose:** Provide STL-like iterator semantics (return raw pointers).
- **Notes:** `rbegin()` and `rend()` appear to assume length ΓëÑ 1.

### simple_array (parallel structure)
- Analogous to `simple_vector` but stores `N*length` elements contiguously.
- `operator[](int indx)` returns pointer to row `indx` (address `contents + N*indx`).
- No STL iterator methods.

## Control Flow Notes
Not inferable from this file. These are utility containers; usage context not visible.

## External Dependencies
- `#include <algorithm.h>` ΓÇô provides `copy()` and `swap()` functions used in constructors and member functions.
- Relies on built-in `new[]` / `delete[]` operators.

## Notes
- **Memory management:** Manual allocation/deallocation; no RAII cleanup on exceptions.
- **Non-const references:** Copy constructor and assignment operator take non-const references, preventing const propagation.
- **No bounds checking:** Index operators do not validate indices.
- **Primitive resizing:** `reallocate()` destroys contents; no grow/shrink semantics like STL vector.
