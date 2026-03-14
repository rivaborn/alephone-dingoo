# Extras/noresnames.cpp

## File Purpose
Command-line utility that strips resource names from all resources in Mac OS Classic resource files. Takes file paths as arguments and iterates through each file, clearing the name field of every resource.

## Core Responsibilities
- Parse command-line file path arguments and validate filename length
- Open resource forks of Mac OS resource files
- Enumerate all resource types within a file
- Enumerate all individual resources for each type
- Retrieve and clear resource names (set to empty string)
- Handle file-level and resource-level errors gracefully

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FSSpec | struct | File system specification for locating and opening files |
| Str255 | typedef | Pascal string (256 bytes max, first byte = length) |
| ResType | typedef | 4-byte resource type identifier |
| Handle | typedef | Mac resource manager handle to loaded resource |

## Global / File-Static State
None.

## Key Functions / Methods

### main
- Signature: `int main(int argc, char **argv)`
- Purpose: Entry point; strips resource names from all resources in files specified via command-line arguments
- Inputs: Command-line arguments (file paths)
- Outputs/Return: Exit code (0)
- Side effects: Modifies resource fork of each input file; prints error messages to stderr
- Calls: `strlen`, `memcpy`, `FSMakeFSSpec`, `FSpOpenResFile`, `SetResLoad`, `ResError`, `Count1Types`, `Get1IndType`, `Count1Resources`, `Get1IndResource`, `GetResInfo`, `SetResInfo`, `ReleaseResource`, `CloseResFile`, `fprintf`
- Notes: Skips files with errors (too-long names, FSMakeFSSpec failures, file open failures); uses `SetResLoad(false/true)` to control whether resources are loaded into memory during enumeration

## Control Flow Notes
Initialization ΓåÆ argument loop ΓåÆ per-file resource fork open ΓåÆ type enumeration ΓåÆ resource enumeration per type ΓåÆ name clearing ΓåÆ cleanup and close ΓåÆ exit. Error handling is per-file with `continue` statements; no fatal errors halt the entire run.

## External Dependencies
- `<Resources.h>`: Mac OS Classic Resource Manager API (`FSMakeFSSpec`, `FSpOpenResFile`, `Count1Types`, `Get1IndResource`, `SetResInfo`, etc.)
- `<stdio.h>`: Standard I/O (`fprintf`)
- `<string.h>`: String utilities (`strlen`, `memcpy`)
