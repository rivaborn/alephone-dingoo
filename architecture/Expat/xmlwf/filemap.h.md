# Expat/xmlwf/filemap.h

## File Purpose
Header file declaring the `filemap` function interface for the Expat XML parser utility. Provides a mechanism to read files and invoke a callback processor on the file contents, with support for both Unicode and ASCII variants.

## Core Responsibilities
- Declare the `filemap` function interface
- Support both wide-character (Unicode) and byte-character (ASCII) file processing
- Define the callback processor function signature for consuming file data
- Abstract file I/O and memory mapping details from callers

## Key Types / Data Structures
None (uses only standard C types: `char`, `wchar_t`, `size_t`, `void *`).

## Global / File-Static State
None.

## Key Functions / Methods

### filemap
- **Signature:**
  - Unicode: `int filemap(const wchar_t *name, void (*processor)(const void *, size_t, const wchar_t *, void *arg), void *arg);`
  - ASCII: `int filemap(const char *name, void (*processor)(const void *, size_t, const char *, void *arg), void *arg);`
- **Purpose:** Maps a file into memory and invokes a callback processor to handle the file contents and metadata.
- **Inputs:**
  - `name`: Filename (wide or narrow character string, depending on `XML_UNICODE` definition)
  - `processor`: Function pointer to callback that processes file data
  - `arg`: Opaque user context passed to the processor callback
- **Outputs/Return:** Integer status code (interpretation not specified in header; likely 0 for success, non-zero for error)
- **Side effects (global state, I/O, alloc):** Opens and reads file; allocates memory for file contents; invokes processor callback with file data
- **Calls (direct calls visible in this file):** None (declaration only)
- **Notes:** The processor callback receives (data pointer, size, filename, user arg); conditional compilation allows single source to support both Unicode and ASCII builds

## Control Flow Notes
This header provides an entry point for file I/O in the `xmlwf` utility. Callers invoke `filemap` with a filename and processor callback; the implementation handles file opening, memory allocation, and callback invocation.

## External Dependencies
- `<stddef.h>` ΓÇö for `size_t` and standard type definitions
- `XML_UNICODE` preprocessor symbol ΓÇö controls which variant is compiled (wide-char vs. byte-char)
