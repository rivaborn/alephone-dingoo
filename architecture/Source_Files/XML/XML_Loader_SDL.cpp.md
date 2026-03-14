# Source_Files/XML/XML_Loader_SDL.cpp

## File Purpose
SDL-based implementation of an XML file parser for the Aleph One game engine. Handles reading and parsing XML configuration files from disk, with error reporting at multiple stages (read, parse, interpretation). Supports both single-file and directory-wide parsing.

## Core Responsibilities
- Provide XML file content to the parent parser class via `GetData()`
- Report read errors, parse errors (with line numbers and filenames), and interpretation errors
- Implement file opening, buffering, and cleanup for single XML files
- Scan and parse all valid XML files in a directory recursively
- Limit error output to prevent log spam (max 7 errors shown)
- Filter out non-XML files (.lua scripts, backup files ending in ~)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `dir_entry` | struct | Represents a directory entry (file/folder metadata); used by `FileSpecifier::ReadDirectory()` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MaxErrorsToShow` | `const int` | file-static | Limits interpretation error output to 7 messages |

## Key Functions / Methods

### GetData
- **Signature:** `bool GetData()`
- **Purpose:** Supplies buffered XML data to the parent `XML_Configure` parser.
- **Inputs:** None (uses private member `data` set by `ParseFile`)
- **Outputs/Return:** `true` if data is valid, `false` if `data == NULL`
- **Side effects:** Sets `Buffer`, `BufLen`, and `LastOne` (inherited fields)
- **Calls:** None (accessor)
- **Notes:** Called by parent class during parsing; returns false if no file was loaded.

### ReportReadError
- **Signature:** `void ReportReadError()`
- **Purpose:** Handle file read failures.
- **Inputs:** None
- **Outputs/Return:** None; prints to stderr and exits.
- **Side effects:** Terminates the program with `exit(1)`
- **Calls:** `fprintf`, `exit`
- **Notes:** Fatal error ΓÇö does not allow recovery.

### ReportParseError
- **Signature:** `void ReportParseError(const char *ErrorString, int LineNumber)`
- **Purpose:** Report XML syntax/structure errors detected by the parser.
- **Inputs:** `ErrorString` (error description), `LineNumber` (where in file)
- **Outputs/Return:** None; prints to stderr.
- **Side effects:** Stderr output; increments internal error counter (inherited)
- **Calls:** `fprintf`
- **Notes:** Includes `FileName` in output for user convenience.

### ReportInterpretError
- **Signature:** `void ReportInterpretError(const char *ErrorString)`
- **Purpose:** Report semantic/interpretation errors in XML content.
- **Inputs:** `ErrorString` (error description)
- **Outputs/Return:** None
- **Side effects:** Stderr output only if error count < 7; calls `GetNumInterpretErrors()` (inherited)
- **Calls:** `GetNumInterpretErrors()`, `fprintf`
- **Notes:** Silently suppresses errors after limit is reached to avoid log spam.

### RequestAbort
- **Signature:** `bool RequestAbort()`
- **Purpose:** Signal to parent parser whether to stop parsing due to excessive errors.
- **Inputs:** None
- **Outputs/Return:** `true` if error count ΓëÑ 7, `false` otherwise
- **Side effects:** None
- **Calls:** `GetNumInterpretErrors()`
- **Notes:** Provides graceful abort instead of running to completion with thousands of errors.

### ParseFile
- **Signature:** `bool ParseFile(FileSpecifier &file_name)`
- **Purpose:** Load and parse a single XML file.
- **Inputs:** `file_name` ΓÇö file to parse
- **Outputs/Return:** `true` if file was opened and processed (even if parse had errors); `false` if file open failed
- **Side effects:** Allocates `data` buffer, reads file contents, triggers parent's `DoParse()`, deallocates buffer
- **Calls:** `FileSpecifier::Open()`, `OpenedFile::GetLength()`, `OpenedFile::Read()`, `DoParse()` (inherited), `fprintf`
- **Notes:** Always attempts cleanup even if read fails; stores filename in `FileName` for error messages.

### ParseDirectory
- **Signature:** `bool ParseDirectory(FileSpecifier &dir)`
- **Purpose:** Scan a directory and parse all valid XML files.
- **Inputs:** `dir` ΓÇö directory to scan
- **Outputs/Return:** `false` if directory read fails; `true` otherwise
- **Side effects:** Calls `ParseFile()` on each non-directory, non-backup, non-Lua file
- **Calls:** `FileSpecifier::ReadDirectory()`, `std::sort()`, `boost::algorithm::ends_with()`, `ParseFile()`
- **Notes:** Skips directories, files ending in `~` (backups), and `.lua` scripts; sorts entries before processing.

## Control Flow Notes
- **Initialization:** Users call either `ParseFile()` or `ParseDirectory()` on a `XML_Loader_SDL` instance.
- **Parse flow:** `ParseFile` or `ParseDirectory` ΓåÆ read file(s) ΓåÆ `GetData()` provides buffer to parent's `DoParse()` ΓåÆ error callbacks invoked during parsing ΓåÆ cleanup.
- **Error accumulation:** Parent class tracks error counts; this file reports and gates output using `MaxErrorsToShow`.
- **Shutdown:** Buffers are deleted in destructor and after each `ParseFile()`.

## External Dependencies
- **Includes:** `cseries.h` (base types, SDL defs), `XML_Loader_SDL.h` (class definition), `FileHandler.h` (file I/O abstractions)
- **STL:** `<vector>`, `<algorithm>` (for `std::sort`)
- **Boost:** `boost::algorithm::string::predicate` (for `ends_with`)
- **Defined elsewhere:** `XML_Configure` base class, `FileSpecifier`, `OpenedFile`, `dir_entry`, `DoParse()`, `GetNumInterpretErrors()`
