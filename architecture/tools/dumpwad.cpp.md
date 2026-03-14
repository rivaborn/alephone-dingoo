# tools/dumpwad.cpp

## File Purpose
Command-line utility to parse and display the contents of Marathon wad files. Reads the wad header, directory metadata, and tag information, then prints a human-readable summary of the level data contained within.

## Core Responsibilities
- Open and read Marathon wad files for inspection
- Parse and validate wad file headers
- Extract and display directory entries with level metadata
- Decode mission flags, environment flags, and entry point information
- Enumerate and list all resource tags within each wad
- Format output for human inspection

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `wad_header` | struct | File header containing version, checksums, directory offset, entry count |
| `directory_entry` | struct | Maps wad index to file offset and data length |
| `directory_data` | struct | Level-specific metadata: mission type, environment, entry points, level name |
| `WadDataType` | typedef | 4-byte tag identifier for resource types |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `temporary` | char[256] | static | String buffer for temporary formatting operations |
| `local_data_dir`, `preferences_dir`, etc. | DirectorySpecifier | global | Directory path configuration (dummy stubs) |

## Key Functions / Methods

### main
- **Signature:** `int main(int argc, char **argv)`
- **Purpose:** Entry point; orchestrates file opening, header parsing, and directory enumeration
- **Inputs:** Command-line arguments (expects a single wad filename)
- **Outputs/Return:** Exit code (0 on success, 1 on error)
- **Side effects:** Prints to stdout; opens and closes file handles
- **Calls:** `open_wad_file_for_reading`, `read_wad_header`, `read_directory_data`, `read_indexed_wad_from_file`, `unpack_directory_data`
- **Notes:** Exits on file open or header read errors; loops through all wads in the file; prints level metadata and tag summaries

### unpack_directory_data
- **Signature:** `static void unpack_directory_data(uint8 *Stream, directory_data *Objects)`
- **Purpose:** Deserialize a single directory_data structure from a byte stream
- **Inputs:** Raw byte stream; pointer to directory_data struct to populate
- **Outputs/Return:** Populates the directory_data object in-place (no return value)
- **Side effects:** Advances the stream pointer
- **Calls:** `StreamToValue`, `StreamToBytes` (packing macros)
- **Notes:** Simple serialization helper; assumes stream is valid and properly positioned

### csprintf
- **Signature:** `char *csprintf(char *buffer, const char *format, ...)`
- **Purpose:** Formatted string composition into caller-provided buffer
- **Inputs:** Target buffer; format string and variadic arguments
- **Outputs/Return:** Pointer to the buffer (for chaining)
- **Side effects:** Writes formatted string to buffer
- **Calls:** `vsprintf`
- **Notes:** Wrapper around standard printf; used for flag-to-text conversions

## Control Flow Notes
**Initialization ΓåÆ File Open ΓåÆ Header Read ΓåÆ Directory Read ΓåÆ Per-Wad Loop:**
1. Validate command-line args; open wad file
2. Read and display wad header (version, checksums, file metadata)
3. Read directory data into memory
4. Iterate over `wad_count` entries:
   - Read directory entry (handles old/new format compatibility)
   - If directory_data is present, decode and print mission/environment/entry-point flags
   - Read wad tags and list them with offsets and sizes
   - Free wad data
5. Exit

No render/update/shutdown cycle; this is a static inspection tool.

## External Dependencies
- **wad.cpp** (included): WAD file reading functions (`open_wad_file_for_reading`, `read_wad_header`, `read_directory_data`, `read_indexed_wad_from_file`, `calculate_directory_offset`)
- **FileHandler_SDL.cpp** (included): File I/O abstraction (`OpenedFile` class, SDL file operations)
- **resource_manager.cpp** (included): Resource file management (included but unused in this tool)
- **crc.cpp** (included): Checksum functions (included for linking)
- **map.h** (header): Map data type definitions (`directory_data`, `directory_entry`, mission/environment/entry-point flag enums)
- **Packing.cpp** (included): Byte serialization macros (`StreamToValue`, `StreamToBytes`)
- **Logging.cpp** (included): Logging support (included for linking)
- **csalerts_sdl.cpp** (included): Error/alert support (dummy stubs for headless operation)
