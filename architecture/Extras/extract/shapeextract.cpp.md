# Extras/extract/shapeextract.cpp

## File Purpose
Command-line utility for extracting shape resources from Macintosh resource files. Reads `.256` resource types from a Mac resource fork and writes structured shape collection data to a binary output file for engine consumption.

## Core Responsibilities
- Parse and validate command-line arguments (source/destination paths)
- Open and manage Macintosh resource files
- Iterate through shape collection IDs and extract resource data
- Write collection headers and resource offsets/lengths to output binary
- Handle resource memory and file I/O errors

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `collection_header` | struct | Metadata for each shape collection (offset, length, offset16, length16); defined in shape_descriptors.h |
| `Str255` | typedef | Macintosh Pascal string (256 bytes max) |
| `Handle` | typedef | Opaque pointer to relocatable memory in Mac Toolbox |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `collection_headers` | `struct collection_header[]` | global | Array of headers for all shape collections; modified during extraction |
| `MAXIMUM_COLLECTIONS` | constant | global | Number of shape collections to extract |

## Key Functions / Methods

### main
- **Signature:** `void main(int argc, char **argv)`
- **Purpose:** Entry point; validates arguments, opens source/destination files, orchestrates extraction
- **Inputs:** Command-line argc/argv (expects `<source_resource_file> <destination_binary_file>`)
- **Outputs/Return:** Exits with code 0 on success, 1 on error
- **Side effects:** Opens/closes Mac resource file, creates binary output file, prints errors to stderr
- **Calls:** `OpenResFile`, `CloseResFile`, `fopen`, `fclose`, `extract_shape_resources`, `ResError`
- **Notes:** Uses Macintosh `c2pstr` to convert C string to Pascal string for resource manager API; errors print ResError() code

### extract_shape_resources
- **Signature:** `static void extract_shape_resources(FILE *stream)`
- **Purpose:** Orchestrates extraction of all shape collections; writes headers, resource data, then rewrites headers with finalized offsets/lengths
- **Inputs:** Open output file stream
- **Outputs/Return:** None; modifies stream via fwrite/fseek
- **Side effects:** Writes to file stream; modifies global `collection_headers` array
- **Calls:** `fwrite`, `fseek`, `add_resource` (twice per collection for standard and 16-bit variants)
- **Notes:** Two-pass approachΓÇöwrites headers once, adds resources, then seeks back and rewrites headers with correct metadata; loops over `MAXIMUM_COLLECTIONS`

### add_resource
- **Signature:** `static void add_resource(short id, long *offset, long *length, FILE *stream)`
- **Purpose:** Retrieves a single shape resource by ID from Mac resource fork and appends to output stream
- **Inputs:** Resource ID, pointers to offset/length fields in header, output stream
- **Outputs/Return:** None; modifies offset/length via pointers, writes to stream
- **Side effects:** Calls Mac Toolbox to lock/release memory; writes to file stream; sets offset/length on error to -1/0
- **Calls:** `GetResource`, `HLock`, `ftell`, `GetHandleSize`, `fwrite`, `ReleaseResource`
- **Notes:** On resource not found, sets offset to -1 and length to 0 (sentinel); uses `*collection` pointer dereference directly in fwrite

## Control Flow Notes
This is a **build-time tool**, not engine runtime code. Runs once during asset pipeline to convert Mac resource fork data into a binary format consumable by the shape engine. The two-pass I/O (write headers, add resources, rewrite headers) ensures resource offsets are finalized before writing metadata.

## External Dependencies
- **Macintosh Toolbox:** `OpenResFile`, `CloseResFile`, `GetResource`, `HLock`, `ReleaseResource`, `ResError`, `GetHandleSize`
- **Standard C:** `FILE`, `fopen`, `fclose`, `fwrite`, `fseek`, `strcpy`, `fprintf`, `exit`
- **Custom headers:** `macintosh_cseries.h` (platform macros/defs), `shape_descriptors.h`, `shape_definitions.h`
- **Mac string utilities:** `c2pstr` (CΓåÆPascal string conversion)
