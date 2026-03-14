# Expat/xmlwf/readfilemap.cpp

## File Purpose
Utility function that reads an entire file from disk into memory and passes its contents to a callback processor. Part of xmlwf (XML Well-Formedness checker) tool infrastructure. Handles file I/O with cross-platform binary mode support and error reporting.

## Core Responsibilities
- Open files in binary read-only mode with POSIX system calls
- Validate files are regular files (not directories or devices)
- Allocate sufficient heap memory for entire file contents
- Read file data into memory with integrity validation
- Invoke caller-provided callback processor with file data
- Release resources (memory, file descriptor) and report status
- Report errors to stderr with context information

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `struct stat` | struct (from `<sys/stat.h>`) | File metadata (size, mode) for validation |

## Global / File-Static State
None.

## Key Functions / Methods

### filemap
- **Signature:** `int filemap(const char *name, void (*processor)(const void *, size_t, const char *, void *arg), void *arg)`
- **Purpose:** Read entire file into memory and invoke callback processor on its contents.
- **Inputs:**
  - `name`: filename (C string)
  - `processor`: callback function pointer accepting (buffer, size, filename, user_arg)
  - `arg`: opaque user data passed to processor
- **Outputs/Return:** 1 on success; 0 on failure (file not found, not regular file, memory exhaustion, I/O error)
- **Side effects:**
  - Allocates heap memory via `malloc()`
  - Opens/closes file descriptor
  - Invokes processor callback with file contents
  - Writes error messages to stderr
- **Calls:** `open()`, `fstat()`, `malloc()`, `read()`, `free()`, `close()`, `perror()`, `fprintf()`, `processor()` (callback)
- **Notes:**
  - Early-return error handling; every failure path closes fd and frees memory
  - Reads entire file into memory (problematic for very large files)
  - Validates file is regular (blocks on directories/devices) via `S_ISREG()` macro
  - Validates read byte count matches expected file size
  - Cross-platform: uses `O_BINARY` macro for portable binary mode on Windows

## Control Flow Notes
One-time utility called during tool startup to load XML document. Not part of a frame/update loopΓÇöexecutes once per file processed, then returns control to caller.

## External Dependencies
- **POSIX system calls:** `open()`, `fstat()`, `read()`, `close()`
- **Standard C library:** `malloc()`, `free()`, `stdio` functions (`perror()`, `fprintf()`)
- **Platform abstraction:** `O_BINARY` macro defined locally to handle Windows binary mode
