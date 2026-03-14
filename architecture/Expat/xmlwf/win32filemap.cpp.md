# Expat/xmlwf/win32filemap.cpp

## File Purpose
Windows-specific file memory mapping implementation for the Expat XML parser. Maps file contents into virtual memory for efficient read access, then invokes a caller-supplied processor callback on the mapped data.

## Core Responsibilities
- Open files via Windows CreateFile API with read-only sequential access
- Create file mappings and map file contents into process virtual address space
- Handle zero-length files as special case (immediate callback with empty data)
- Enforce 2GB file size limit
- Clean up handles and mappings in all code paths
- Report Windows API errors to stderr with system error messages
- Support both ANSI and Unicode file names (conditional via XML_UNICODE macro)

## Key Types / Data Structures
None (uses Windows API types: HANDLE, DWORD, void pointers).

## Global / File-Static State
None.

## Key Functions / Methods

### filemap
- **Signature:** `int filemap(const TCHAR *name, void (*processor)(const void *, size_t, const TCHAR *, void *arg), void *arg)`
- **Purpose:** Map a file into memory and invoke a callback processor on its contents.
- **Inputs:**
  - `name`: File path (TCHARΓÇöchar or wchar_t per XML_UNICODE)
  - `processor`: Callback function to process file data
  - `arg`: User context forwarded to processor
- **Outputs/Return:** 1 on success, 0 on failure.
- **Side effects:** File I/O (open, map, close); memory mapping; calls processor with mapped address; stderr output on error.
- **Calls:** CreateFile, GetFileSize, CreateFileMapping, MapViewOfFile, UnmapViewOfFile, CloseHandle, win32perror, processor (callback).
- **Notes:** 
  - Zero-length files bypass CreateFileMapping (which fails on empty files); processor called with static '\0' instead.
  - 32-bit size limit (2GB max).
  - Sequential read access flag set on open.
  - Resource cleanup (UnmapViewOfFile, CloseHandle) occurs on all paths.

### win32perror (static)
- **Signature:** `static void win32perror(const TCHAR *s)`
- **Purpose:** Format and print the last Windows system error to stderr.
- **Inputs:** `s`: Context string (typically filename).
- **Outputs/Return:** void.
- **Side effects:** Allocates and frees buffer; writes to stderr.
- **Calls:** FormatMessage (with FORMAT_MESSAGE_ALLOCATE_BUFFER), GetLastError, _ftprintf, LocalFree.
- **Notes:** Falls back to generic "unknown Windows error" message if FormatMessage fails.

## Control Flow Notes
Entry point for the xmlwf tool's file reading workflow. Filemap is a utility layer between the caller and Windows file I/O, abstracting resource management. No game engine integration apparent.

## External Dependencies
- `<windows.h>` ΓÇö Windows API (CreateFile, CreateFileMapping, MapViewOfFile, CloseHandle, FormatMessage, GetLastError, LocalFree)
- `<stdio.h>` ΓÇö _ftprintf
- `<tchar.h>` ΓÇö TCHAR, _T (Unicode/ANSI abstraction)
- `"filemap.h"` ΓÇö Function declaration
