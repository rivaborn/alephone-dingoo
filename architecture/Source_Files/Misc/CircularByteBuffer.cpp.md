# Source_Files/Misc/CircularByteBuffer.cpp

## File Purpose
Implementation of a circular queue optimized for byte-level buffering, supporting both traditional copy-based and zero-copy interfaces for enqueueing and peeking data. Handles wraparound automatically when data spans the buffer's circular boundary.

## Core Responsibilities
- Enqueue variable-length byte sequences with automatic wraparound handling
- Peek at buffered bytes without advancing read position
- Provide zero-copy access paths via direct buffer pointers for performance-critical code
- Split logical byte ranges into up-to-two contiguous chunks when they cross the buffer seam
- Assert preconditions to catch buffer overflow/underflow in debug builds

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CircularByteBufferBase` | typedef (via header) | Alias for `CircularQueue<char>`; inherited base class |
| `std::pair<unsigned int, unsigned int>` | std struct | Returns (first_chunk_size, second_chunk_size) from `splitIntoChunks()` |

## Global / File-Static State
None.

## Key Functions / Methods

### splitIntoChunks
- **Signature:** `static std::pair<unsigned int, unsigned int> splitIntoChunks(unsigned int inByteCount, unsigned int inStartingIndex, unsigned int inQueueSize)`
- **Purpose:** Calculate how many bytes fit in the first contiguous segment before wraparound, and how many must wrap to the buffer start.
- **Inputs:** `inByteCount` (total bytes to place/read), `inStartingIndex` (read or write index), `inQueueSize` (total buffer capacity)
- **Outputs/Return:** Pair of (bytes_in_first_segment, bytes_in_second_segment)
- **Side effects:** None
- **Calls:** `std::min()`
- **Notes:** Second chunk size is always `inByteCount - firstChunkSize`. If second is 0, no wraparound needed.

### enqueueBytes
- **Signature:** `void enqueueBytes(const void* inBytes, unsigned int inByteCount)`
- **Purpose:** Copy bytes from caller's buffer into the circular queue.
- **Inputs:** `inBytes` (source buffer), `inByteCount` (number of bytes to copy)
- **Outputs/Return:** None
- **Side effects:** Advances write index via `advanceWriteIndex()`, modifies `mData`
- **Calls:** `splitIntoChunks()`, `memcpy()` (up to twice), `getRemainingSpace()`, `advanceWriteIndex()`
- **Notes:** Asserts `inByteCount <= getRemainingSpace()` to catch overflow; handles split copies with separate memcpy calls

### enqueueBytesNoCopyStart
- **Signature:** `void enqueueBytesNoCopyStart(unsigned int inByteCount, void** outFirstBytes, unsigned int* outFirstByteCount, void** outSecondBytes, unsigned int* outSecondByteCount)`
- **Purpose:** Begin a no-copy enqueue by returning pointers to where caller should write bytes.
- **Inputs:** `inByteCount` (planned write size), out-pointers (all optional, may be NULL)
- **Outputs/Return:** Fills out-parameters with pointers and byte counts for up to two buffer regions
- **Side effects:** None; write index is *not* advanced until `enqueueBytesNoCopyFinish()` is called
- **Calls:** `splitIntoChunks()`, `getWriteIndex()`
- **Notes:** Allows caller to use writev()-style I/O directly into buffer; second pointer/count are NULL/0 if no wraparound

### enqueueBytesNoCopyFinish
- **Signature:** `void enqueueBytesNoCopyFinish(unsigned int inActualByteCount)`
- **Purpose:** Finalize a no-copy enqueue after caller has written data to the pointers from `enqueueBytesNoCopyStart()`.
- **Inputs:** `inActualByteCount` (actual bytes written; may be Γëñ planned count)
- **Outputs/Return:** None
- **Side effects:** Advances write index by `inActualByteCount` via `advanceWriteIndex()`
- **Calls:** `advanceWriteIndex()`
- **Notes:** Caller may write fewer bytes than originally planned; must be paired with prior `enqueueBytesNoCopyStart()`

### peekBytes
- **Signature:** `void peekBytes(void* outBytes, unsigned int inByteCount)`
- **Purpose:** Copy bytes from the queue into caller's buffer without advancing read position.
- **Inputs:** `outBytes` (destination buffer), `inByteCount` (number of bytes to copy)
- **Outputs/Return:** Fills `outBytes` with up to `inByteCount` bytes
- **Side effects:** None
- **Calls:** `splitIntoChunks()`, `memcpy()` (up to twice), `getCountOfElements()`, `mReadIndex`
- **Notes:** Asserts `inByteCount <= getCountOfElements()`; handles split peeks with separate memcpy calls

### peekBytesNoCopy
- **Signature:** `void peekBytesNoCopy(unsigned int inByteCount, const void** outFirstBytes, unsigned int* outFirstByteCount, const void** outSecondBytes, unsigned int* outSecondByteCount)`
- **Purpose:** Peek at queued bytes via direct pointers without copying or advancing read index.
- **Inputs:** `inByteCount` (planned read size), out-pointers (all optional, may be NULL)
- **Outputs/Return:** Fills out-parameters with const pointers and byte counts for up to two buffer regions
- **Side effects:** None
- **Calls:** `splitIntoChunks()`, `getReadIndex()`
- **Notes:** Allows caller to use readv()-style I/O directly from buffer; second pointer/count are NULL/0 if no wraparound; caller still must call `dequeue()` separately to advance read index

## Control Flow Notes
This is a data structure utility, not tied to the game loop. Typical usage: network code, disk I/O, or inter-system buffering that calls enqueue/peek/dequeue methods as needed. No implicit frame-to-frame control flow.

## External Dependencies
- `<algorithm>` ΓÇö `std::min()`
- `<utility>` ΓÇö `std::pair`
- `"cseries.h"` ΓÇö `assert()` macro
- `"CircularQueue.h"` ΓÇö base class `CircularQueue<T>` (methods like `advanceWriteIndex()`, `getRemainingSpace()`, `getCountOfElements()`, `getWriteIndex()`, `getReadIndex()` are inherited)
