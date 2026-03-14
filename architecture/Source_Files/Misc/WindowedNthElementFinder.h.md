# Source_Files/Misc/WindowedNthElementFinder.h

## File Purpose
A template data structure that maintains a fixed-size sliding window of recently inserted elements and provides efficient queries for the nth smallest or nth largest element. Automatically evicts the oldest element when the window reaches capacity.

## Core Responsibilities
- Manage a circular buffer of recent elements (insertion order via `CircularQueue`)
- Maintain sorted order of buffered elements (via `std::multiset`)
- Support O(n) lookup of nth smallest/largest within the window
- Automatically evict oldest elements when the window is full
- Support resizing the window after construction

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `tElementType` | template parameter | Generic element type |
| `mQueue` | `CircularQueue<tElementType>` | FIFO buffer tracking insertion order |
| `mSortedElements` | `std::multiset<tElementType>` | Sorted container for efficient nth-element queries |

## Global / File-Static State
None.

## Key Functions / Methods

### WindowedNthElementFinder()
- Signature: `WindowedNthElementFinder()`
- Purpose: Default constructor; creates an empty finder with zero window size
- Inputs: None
- Outputs/Return: Instance with `mQueue` initialized to size 0
- Side effects: None
- Calls: `CircularQueue` default constructor
- Notes: Window size must be set via `reset()` before use

### WindowedNthElementFinder(unsigned int)
- Signature: `explicit WindowedNthElementFinder(unsigned int inWindowSize)`
- Purpose: Construct with a specified window capacity
- Inputs: `inWindowSize` ΓÇô max elements to buffer
- Outputs/Return: Instance with allocated circular queue
- Side effects: Allocates dynamic memory via `CircularQueue`
- Calls: `CircularQueue` sized constructor
- Notes: Enables RAII initialization with a known window size

### reset(unsigned int)
- Signature: `void reset(unsigned int inWindowSize)`
- Purpose: Clear all elements and resize the window
- Inputs: `inWindowSize` ΓÇô new window capacity
- Outputs/Return: None
- Side effects: Deallocates/reallocates `mQueue`, clears `mSortedElements`
- Calls: `mQueue.reset()`, `mSortedElements.clear()`
- Notes: Used for reinitializing the finder with a different window size

### insert(const tElementType&)
- Signature: `void insert(const tElementType& inNewElement)`
- Purpose: Add an element to the window; evict oldest if window is full
- Inputs: `inNewElement` ΓÇô element to insert
- Outputs/Return: None
- Side effects: Modifies both `mQueue` and `mSortedElements`; may deallocate old element
- Calls: `window_full()`, `mQueue.peek()`, `mQueue.dequeue()`, `mSortedElements.erase()`, `mQueue.enqueue()`, `mSortedElements.insert()`
- Notes: O(log n) due to multiset operations; assumes `tElementType` is copy-constructible and comparable

### nth_smallest_element(unsigned int)
- Signature: `const tElementType& nth_smallest_element(unsigned int n)`
- Purpose: Return the nth smallest element in the current window (0-indexed)
- Inputs: `n` ΓÇô zero-based index (0 = smallest)
- Outputs/Return: Const reference to the nth smallest element
- Side effects: None
- Calls: `size()`, `mSortedElements.begin()` (const iterator)
- Notes: O(n) linear iteration from smallest; asserts `n < size()`

### nth_largest_element(unsigned int)
- Signature: `const tElementType& nth_largest_element(unsigned int n)`
- Purpose: Return the nth largest element in the current window (0-indexed)
- Inputs: `n` ΓÇô zero-based index (0 = largest)
- Outputs/Return: Const reference to the nth largest element
- Side effects: None
- Calls: `size()`, `mSortedElements.rbegin()` (const reverse iterator)
- Notes: O(n) reverse iteration from largest; asserts `n < size()`

## Control Flow Notes
This is a passive data structureΓÇönot integrated into a game loop. Operations are query-based:
- **Initialization**: Construct with `WindowedNthElementFinder(size)` or call `reset()`.
- **Update phase**: Call `insert()` repeatedly with new elements.
- **Query phase**: Call `nth_smallest_element()` or `nth_largest_element()` to inspect state.
- No rendering, shutdown, or frame-dependent behavior.

## External Dependencies
- **`CircularQueue.h`**: Provides FIFO buffering with modulo-based wrap-around indexing.
- **`<multiset>`** (STL): Maintains sorted order; requires template type `tElementType` to be comparable (operator `<`).
- **`assert`** macro: Runtime bounds checks in `nth_smallest_element()`, `nth_largest_element()`, and `CircularQueue` methods.
- **Standard C++ template instantiation**: Entire class is template-based; no explicit instantiations in this file.
