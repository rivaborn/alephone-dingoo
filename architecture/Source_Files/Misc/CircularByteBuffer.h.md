# Source_Files/Misc/CircularByteBuffer.h

## File Purpose
Provides a circular byte buffer (queue) with safe wraparound handling for mass byte operations. Offers both copy-based and zero-copy interfaces to minimize unnecessary data copying during enqueue/dequeue operations, useful for network/IO buffering in the game engine.

## Core Responsibilities
- Implement a byte-oriented circular queue extending `CircularQueue<char>`
- Provide bulk peek/enqueue operations with automatic wraparound handling
- Expose zero-copy read/write interfaces via pointer-pair returns for integration with `writev()`/`readv()`-style operations
- Utility function to calculate wraparound boundaries
- Ensure safe multi-chunk access when data spans the buffer's circular seam

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `CircularByteBufferBase` | typedef | Alias for `CircularQueue<char>` |

## Global / File-Static State
None.

## Key Functions / Methods

### peekBytes
- **Signature:** `void peekBytes(void* outBytes, unsigned int inByteCount)`
- **Purpose:** Copy bytes from buffer without removing them, handling wraparound transparently.
- **Inputs:** destination buffer pointer, byte count to peek
- **Outputs:** Data copied to `outBytes`
- **Side effects:** None (read state unchanged)
- **Calls:** Not visible in header
- **Notes:** Caller must ensure `inByteCount <= getCountOfElements()`

### enqueueBytes
- **Signature:** `void enqueueBytes(const void* inBytes, unsigned int inByteCount)`
- **Purpose:** Add bytes to buffer with automatic wraparound handling.
- **Inputs:** source buffer pointer, byte count to enqueue
- **Outputs:** None
- **Side effects:** Advances write index, modifies buffer contents
- **Notes:** Caller must ensure `inByteCount <= getRemainingSpace()`

### peekBytesNoCopy
- **Signature:** `void peekBytesNoCopy(unsigned int inByteCount, const void** outFirstBytes, unsigned int* outFirstByteCount, const void** outSecondBytes, unsigned int* outSecondByteCount)`
- **Purpose:** Expose raw buffer pointers for reading without copying, accounting for wraparound.
- **Inputs:** desired byte count
- **Outputs:** Two pointer/length pairs (first and second chunk of contiguous memory)
- **Side effects:** None; read index not advanced until `dequeue()` called
- **Notes:** Second chunk is NULL/0 if data doesn't span wraparound; sum of counts = `inByteCount`

### enqueueBytesNoCopyStart
- **Signature:** `void enqueueBytesNoCopyStart(unsigned int inByteCount, void** outFirstBytes, unsigned int* outFirstByteCount, void** outSecondBytes, unsigned int* outSecondByteCount)`
- **Purpose:** Begin zero-copy enqueue by exposing write destinations.
- **Inputs:** planned byte count
- **Outputs:** Two pointer/length pairs for write locations
- **Side effects:** None; write index advanced only in `enqueueBytesNoCopyFinish()`
- **Notes:** Must be paired with `enqueueBytesNoCopyFinish()`; actual write can be less than planned

### enqueueBytesNoCopyFinish
- **Signature:** `void enqueueBytesNoCopyFinish(unsigned int inActualByteCount)`
- **Purpose:** Finalize zero-copy enqueue by advancing write index.
- **Inputs:** actual bytes written
- **Outputs:** None
- **Side effects:** Advances write index by `inActualByteCount`
- **Notes:** Separates pointer exposure from index advancement for data safety

### splitIntoChunks (static)
- **Signature:** `static std::pair<unsigned int, unsigned int> splitIntoChunks(unsigned int inByteCount, unsigned int inStartingIndex, unsigned int inQueueSize)`
- **Purpose:** Calculate how a given byte count splits across a circular wraparound boundary.
- **Inputs:** byte count needed, starting index, queue size
- **Outputs:** `std::pair` where `first + second == inByteCount`
- **Notes:** Static utility; usable outside class; second is 0 if no wraparound needed

## Control Flow Notes
Data structure utility; not part of game loop. Likely used for network packet buffering, file I/O staging, or inter-thread communication where circular buffering is beneficial.

## External Dependencies
- `<utility>` ΓÇô `std::pair`
- `CircularQueue.h` ΓÇô template base class `CircularQueue<T>` with index management and wraparound arithmetic
