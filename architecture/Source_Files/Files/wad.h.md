# Source_Files/Files/wad.h

## File Purpose
Header file defining the binary WAD file format and API for reading, writing, and manipulating WAD containers (used in Marathon/Aleph One to store levels, sprites, textures, and other game data). Provides file I/O abstractions, in-memory data structures, and operations for loading, extracting, and modifying tagged game assets.

## Core Responsibilities
- Define binary WAD file format structures (headers, directories, entries) with version compatibility
- Declare functions for opening, closing, and navigating WAD files
- Provide API for reading and extracting tagged data from WADs
- Support creating, writing, and modifying WAD files in-place
- Validate file integrity via checksums and support WAD inheritance (parent-child relationships)
- Manage memory allocation for loaded WAD structures with between-levels mode control
- Abstract file I/O through FileSpecifier and OpenedFile classes

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| wad_header | struct | 128-byte binary header; stores version, filename, checksum, directory offset, WAD count, and metadata |
| directory_entry | struct | 10-byte directory record; maps offset and length with index for in-place modification (supersedes old_directory_entry) |
| old_directory_entry | struct | 8-byte legacy directory record; offset and length only |
| entry_header | struct | 16-byte header for individual tagged entries; stores tag ID, next offset, length, and expansion offset |
| old_entry_header | struct | 12-byte legacy entry header (without expansion offset field) |
| tag_data | struct | In-memory tagged data block; holds tag type, data pointer, length, and patch offset |
| wad_data | struct | In-memory WAD representation; contains tag count, read-only flag, and dynamic tag_data array |
| WadDataType | typedef | uint32 for 4-character tag identifiers |

## Global / File-Static State
None.

## Key Functions / Methods

### open_wad_file_for_reading / open_wad_file_for_writing
- Signature: `bool open_wad_file_for_reading(FileSpecifier& File, OpenedFile& OFile);` / `bool open_wad_file_for_writing(FileSpecifier& File, OpenedFile& OFile);`
- Purpose: Open WAD file for read or write operations
- Inputs: FileSpecifier (file to open), OpenedFile reference (output)
- Outputs/Return: bool (true if successful)
- Side effects: Opens file handle, initializes OFile
- Calls: (defined elsewhere)
- Notes: Separate functions for read/write modes; OFile is output parameter

### read_indexed_wad_from_file
- Signature: `struct wad_data *read_indexed_wad_from_file(OpenedFile& OFile, struct wad_header *header, short index, bool read_only);`
- Purpose: Load a single WAD (by index) from file into memory
- Inputs: Opened file, WAD header info, WAD index, read-only flag
- Outputs/Return: Pointer to allocated wad_data structure
- Side effects: Allocates memory, reads from file
- Calls: (defined elsewhere)
- Notes: read_only flag prevents modification in memory

### extract_type_from_wad
- Signature: `void *extract_type_from_wad(struct wad_data *wad, WadDataType type, size_t *length);`
- Purpose: Retrieve a specific tagged entry from a loaded WAD
- Inputs: wad_data pointer, tag type to extract, length output parameter
- Outputs/Return: Void pointer to tag data (or NULL if not found), length written to output
- Side effects: None (read-only access)
- Calls: (defined elsewhere)
- Notes: Returns pointer into existing WAD memory, not a copy

### write_wad
- Signature: `bool write_wad(OpenedFile& OFile, struct wad_header *file_header, struct wad_data *wad, int32 offset);`
- Purpose: Write WAD data to file at specified offset
- Inputs: Opened file for writing, file header, wad_data, file offset
- Outputs/Return: bool (true if successful)
- Side effects: Writes to disk
- Calls: (defined elsewhere)
- Notes: Called after create/modify operations to persist changes

### append_data_to_wad / remove_tag_from_wad
- Signature: `struct wad_data *append_data_to_wad(struct wad_data *wad, WadDataType type, void *data, size_t size, size_t offset);` / `void remove_tag_from_wad(struct wad_data *wad, WadDataType type);`
- Purpose: Add or remove a tagged entry from in-memory WAD
- Inputs: wad_data, tag type, data buffer and size (for append), patch offset (for append)
- Outputs/Return: Updated wad_data pointer (may differ if reallocated); void (remove)
- Side effects: Reallocates tag_data array; modifies WAD structure
- Calls: (defined elsewhere)
- Notes: append_data_to_wad may return different pointer due to reallocation

### get_flat_data / inflate_flat_data
- Signature: `void *get_flat_data(FileSpecifier& File, bool use_union, short wad_index);` / `struct wad_data *inflate_flat_data(void *data, struct wad_header *header);`
- Purpose: Serialize WAD to contiguous buffer (flat) and deserialize back
- Inputs: File and options (get_flat_data); flat buffer and header (inflate_flat_data)
- Outputs/Return: Flat buffer pointer; wad_data pointer
- Side effects: Memory allocation; calls free_wad() for cleanup in inflate
- Calls: (defined elsewhere)
- Notes: Complementary pair for transferring WAD data

### calculate_and_store_wadfile_checksum / read_wad_file_checksum
- Signature: `void calculate_and_store_wadfile_checksum(OpenedFile& OFile);` / `uint32 read_wad_file_checksum(FileSpecifier& File);`
- Purpose: Compute and persist file integrity checksum; read stored checksum
- Inputs: Opened file (calculate); FileSpecifier (read)
- Outputs/Return: void (calculate); uint32 checksum (read)
- Side effects: calculate_and_store modifies file on disk
- Calls: (defined elsewhere)
- Notes: Validates file hasn't been corrupted or modified externally

### SetBetweenlevels / IsBetweenLevels
- Signature: `void SetBetweenlevels(bool _BetweenLevels);` / `bool IsBetweenLevels();`
- Purpose: Toggle memory allocator mode to separate level data from runtime-loaded models
- Inputs: bool (SetBetweenlevels)
- Outputs/Return: void (Set); bool (Is)
- Side effects: Affects global allocator behavior
- Calls: (defined elsewhere)
- Notes: Prevents in-game model loads from interfering with level data structures

**Trivial helpers** summarized: `create_wadfile`, `close_wad_file`, `read_wad_header`, `number_of_wads_in_file`, `free_wad`, `create_empty_wad`, `fill_default_wad_header`, `write_wad_header`, `write_directorys`, `read_directory_data`, `get_indexed_directory_data`, `calculate_wad_length`, `get_size_of_directory_data`, `wad_file_has_checksum`, `wad_file_has_parent_checksum`, `read_wad_file_parent_checksum`, `find_wad_file_that_has_checksum`, `find_file_with_modification_date`, `set_indexed_directory_offset_and_length`, `dump_wad`.

## Control Flow Notes
Typical WAD lifecycle: *Initialize typecodes* ΓåÆ *Open file* (read/write mode) ΓåÆ *Read header* to get metadata ΓåÆ *Load WAD by index* into memory ΓåÆ *Extract tagged data* (level geometry, sprites, etc.) ΓåÆ *Optionally modify* (append/remove tags) ΓåÆ *Write to file* (if modified) + *Calculate checksum* ΓåÆ *Close file* ΓåÆ *Free memory*. Between-levels mode (SetBetweenlevels) controls whether allocations interfere with active level dataΓÇöenabled by default at level transitions.

## External Dependencies
- `#include "tags.h"` ΓÇö Defines Typecode enum, tag constants (FOUR_CHARS_TO_INT macro), file type codes, typecode accessors
- Forward declarations: `class FileSpecifier;` and `class OpenedFile;` ΓÇö Cross-platform file abstraction (implementation elsewhere)
- Implicit: Standard C types (int16, int32, uint32, byte, char) from cstypes.h (included via tags.h)
