# Expat/xmlwf/unixfilemap.cpp

## File Purpose
Implements Unix/POSIX file memory-mapping for the Expat XML parser's well-formedness checker (xmlwf). Opens a file, validates it, memory-maps its contents via `mmap()`, and invokes a processor callback with the mapped data.

## Core Responsibilities
- Open and validate file existence and type (regular file check via `fstat` and `S_ISREG`)
- Memory-map file contents using POSIX `mmap()` for efficient zero-copy access
- Invoke a processor callback function with mapped memory, byte count, filename, and user context
- Clean up resources: unmap memory and close file descriptor
- Provide error reporting via `perror()` and `fprintf()` for diagnostics
- Return success/failure status to caller

## Key Types / Data Structures
None defined in this file. Uses standard POSIX types: `struct stat`, file descriptors (int).

## Global / File-Static State
None

## Key Functions / Methods

### filemap
- **Signature:** `int filemap(const char *name, void (*processor)(const void *, size_t, const char *, void *arg), void *arg)`
- **Purpose:** Memory-map a file and invoke a callback processor with its contents.
- **Inputs:**
  - `name`: Filename (C string)
  - `processor`: Callback function pointer taking (mapped buffer, size, filename, user arg)
  - `arg`: Opaque user context passed to processor
- **Outputs/Return:** Integer (1 = success, 0 = failure)
- **Side effects:**
  - I/O: `open()`, `fstat()`, `close()`
  - Memory: `mmap()` and `munmap()` kernel page mapping
  - Error output: calls `perror()` and `fprintf(stderr, ...)`
- **Calls:** `open()`, `fstat()`, `close()`, `mmap()`, `munmap()`, `perror()`, `fprintf()`
- **Notes:**
  - Returns 0 (failure) if file open fails, stat fails, file is not regular, or mmap fails
  - File descriptor is always closed before return (even on error)
  - Processor callback is responsible for consuming the mapped data; this function doesn't interpret it
  - Uses `MAP_FILE | MAP_PRIVATE` flags (MAP_FILE is 0 on most systems; MAP_PRIVATE means copy-on-write)
  - `MAP_FILE` fallback for portability: defined as 0 if the system doesn't define it

## Control Flow Notes
**Initialization/Main sequence:** This is a utility called during XML parsing setup. The flow is: open ΓåÆ validate ΓåÆ map ΓåÆ process ΓåÆ cleanup. Not part of a frame/render loop; purely synchronous file I/O for parser input.

## External Dependencies
- **System headers:** `<sys/types.h>`, `<sys/mman.h>`, `<sys/stat.h>`, `<fcntl.h>`, `<errno.h>`, `<stdio.h>`, `<string.h>`
- **Project header:** `filemap.h` (declares function prototype; conditionally uses `wchar_t` for Unicode builds)
- **External symbols:** `open()`, `close()`, `fstat()`, `mmap()`, `munmap()`, `S_ISREG()` (macro), `perror()`, `fprintf()` (POSIX/libc)
