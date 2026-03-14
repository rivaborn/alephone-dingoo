# Source_Files/Files/wad.cpp

## File Purpose
Implements WAD (Where's All the Data) file I/O for the Marathon/Aleph One game engine. Provides serialization and deserialization of game resources (maps, textures, sounds, etc.) packaged into binary WAD archives, supporting multiple file format versions with backwards compatibility.

## Core Responsibilities
- Read WAD file headers and validate version/format compatibility
- Load indexed WADs from disk into in-memory `wad_data` structures (read-only or modifiable)
- Write WAD data with entry headers and directory information back to files
- Manage tag-based data queries (extract specific resource types from loaded WADs)
- Handle memory allocation strategy (level-transition vs. standard malloc)
- Support binary packing/unpacking with endianness conversion
- Calculate and verify WAD file checksums (CRC)
- Serialize/flatten WADs for network transfer
- Support multiple WAD file versions (Marathon 1 ΓåÆ Marathon Infinity)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `wad_header` | struct | 128-byte file header (version, checksum, directory offset, WAD count) |
| `wad_data` | struct | In-memory WAD representation (tag array, read-only flag) |
| `tag_data` | struct | Individual tag entry (type, data pointer, length, offset) |
| `directory_entry` / `old_directory_entry` | struct | File directory metadata (offset, length, index) |
| `entry_header` / `old_entry_header` | struct | Per-tag header in WAD (tag ID, next offset, length) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `internal_data` | `struct wad_internal_data*[MAXIMUM_OPEN_WADFILES]` | static | Tracks up to 3 simultaneously open WAD file handles |
| `BetweenLevels` | bool | static | Determines allocation strategy: `true` uses `level_transition_malloc()`, `false` uses standard `malloc()` |

## Key Functions / Methods

### read_wad_header
- **Signature:** `bool read_wad_header(OpenedFile& OFile, struct wad_header *header)`
- **Purpose:** Read and validate WAD file header from disk.
- **Inputs:** Opened file reference; pointer to header struct to populate.
- **Outputs/Return:** `true` if successful; header populated; `false` on read error or invalid version.
- **Side effects:** Calls `set_game_error()` on failure.
- **Calls:** `read_from_file()`, `unpack_wad_header()`, `set_game_error()`.
- **Notes:** Validates version Γëñ CURRENT_WADFILE_VERSION, data_version Γëñ 2, wad_count ΓëÑ 1. Unpacks big-endian stream format.

### read_indexed_wad_from_file
- **Signature:** `struct wad_data *read_indexed_wad_from_file(OpenedFile& OFile, struct wad_header *header, short index, bool read_only)`
- **Purpose:** Load a single indexed WAD from file into memory; main entry point for WAD loading.
- **Inputs:** Opened file; header (with directory offset); WAD index; read-only flag.
- **Outputs/Return:** Pointer to allocated `wad_data` struct; `NULL` on error.
- **Side effects:** Allocates memory via `level_transition_malloc()` or `malloc()`. If read-only, holds reference to file buffer; if modifiable, copies data and frees raw buffer.
- **Calls:** `size_of_indexed_wad()`, `read_indexed_wad_from_file_into_buffer()`, `convert_wad_from_raw()`, `convert_wad_from_raw_modifiable()`, `set_game_error()`.
- **Notes:** Adds padding for Marathon 1 compatibility (shorter entry headers). Memory requirement is ~2├ù level size for conversion.

### extract_type_from_wad
- **Signature:** `void *extract_type_from_wad(struct wad_data *wad, WadDataType type, size_t *length)`
- **Purpose:** Query a loaded WAD for a specific tag type (e.g., POLYGON_TAG, LIGHTSOURCE_TAG).
- **Inputs:** Pointer to `wad_data`; desired tag type; pointer to length output variable.
- **Outputs/Return:** Pointer to tag data; length set to tag size; `NULL` if tag not found.
- **Side effects:** None.
- **Calls:** Linear search through `wad->tag_data[]`.
- **Notes:** Returns pointer into WAD buffer; do not free.

### write_wad
- **Signature:** `bool write_wad(OpenedFile& OFile, struct wad_header *file_header, struct wad_data *wad, int32 offset)`
- **Purpose:** Write WAD data with entry headers to file at specified offset.
- **Inputs:** Opened file for writing; header (determines entry header format); WAD data; file offset.
- **Outputs/Return:** `true` on success.
- **Side effects:** Writes entry headers and data sequentially to file; updates running file offset.
- **Calls:** `get_entry_header_length()`, `pack_old_entry_header()`, `pack_entry_header()`, `write_to_file()`.
- **Notes:** Sets `next_offset` to 0 for last entry; handles both old (12-byte) and new (16-byte) entry header formats.

### append_data_to_wad
- **Signature:** `struct wad_data *append_data_to_wad(struct wad_data *wad, WadDataType type, void *data, size_t size, size_t offset)`
- **Purpose:** Add or replace a tag in a modifiable WAD.
- **Inputs:** WAD struct; tag type; data pointer; size; offset hint (for in-place expansion).
- **Outputs/Return:** Updated `wad` pointer.
- **Side effects:** Reallocates tag array if appending new tag; frees old data if replacing; copies new data.
- **Calls:** `malloc()`, `free()`, `objlist_clear()`, `objlist_copy()`, `memcpy()`, `alert_user()`.
- **Notes:** Asserts WAD is not read-only; aborts on allocation failure.

### get_flat_data / inflate_flat_data
- **Signature:**
  - `void *get_flat_data(FileSpecifier& File, bool use_union, short wad_index)`
  - `struct wad_data *inflate_flat_data(void *data, struct wad_header *header)`
- **Purpose:** Serialize/deserialize WAW for network transfer. `get_flat_data()` encapsulates a WAD with header into a single byte buffer; `inflate_flat_data()` reconstructs it.
- **Inputs:** File spec and index (get); flat data pointer and header (inflate).
- **Outputs/Return:** Flat data buffer pointer (get); reconstructed `wad_data` (inflate).
- **Side effects:** Allocates buffer with magic cookie and encapsulated header prepended.
- **Calls:** `malloc()`, `free()`, `read_indexed_wad_from_file_into_buffer()`, `unpack_wad_header()`, `convert_wad_from_raw()`.
- **Notes:** Uses `CURRENT_FLAT_MAGIC_COOKIE` for validation; used for multiplayer map transfer.

### calculate_and_store_wadfile_checksum
- **Signature:** `void calculate_and_store_wadfile_checksum(OpenedFile& OFile)`
- **Purpose:** Compute CRC-32 checksum for entire WAD file and store in header.
- **Inputs:** Opened file for reading and writing.
- **Outputs/Return:** None.
- **Side effects:** Reads header, zeros checksum field, writes header, calculates CRC, writes CRC back.
- **Calls:** `read_wad_header()`, `write_wad_header()`, `calculate_crc_for_opened_file()`.
- **Notes:** Ensures checksum field is cleared before computation to avoid circular dependency.

### convert_wad_from_raw (private)
- **Signature:** `static struct wad_data *convert_wad_from_raw(struct wad_header *header, uint8 *data, int32 wad_start_offset, int32 raw_length)`
- **Purpose:** Parse raw WAD binary data into in-memory `wad_data` structure (read-only mode).
- **Inputs:** Header (for entry header size); raw buffer; offset within buffer; WAD length.
- **Outputs/Return:** Allocated `wad_data` struct; tag pointers reference the input buffer.
- **Side effects:** Allocates `wad_data` and `tag_data` array; stores pointer to input buffer in `read_only_data` field.
- **Calls:** `malloc()`, `obj_clear()`, `count_raw_tags()`, `objlist_clear()`, `unpack_entry_header()`.
- **Notes:** Zero-copy for data; entry headers are unpacked in-place from buffer. Does not free raw buffer.

### Packing/Unpacking Functions (private)
- **Signatures:** `unpack_wad_header()`, `pack_wad_header()`, `unpack_directory_entry()`, `pack_directory_entry()`, `unpack_entry_header()`, `pack_entry_header()`, etc.
- **Purpose:** Convert between big-endian packed stream format and native C struct format.
- **Inputs:** Byte stream pointer; object array; count.
- **Outputs/Return:** Updated stream pointer (advanced past consumed/written bytes).
- **Calls:** `StreamToValue()`, `ValueToStream()`, `StreamToBytes()`, `BytesToStream()` (from Packing.h).
- **Notes:** Support both old (Marathon 1) and new (Marathon Infinity) format variants.

## Control Flow Notes
**Load path:** `open_wad_file_for_reading()` ΓåÆ `read_wad_header()` ΓåÆ `read_indexed_wad_from_file()` ΓåÆ `convert_wad_from_raw()` or `convert_wad_from_raw_modifiable()` ΓåÆ application queries via `extract_type_from_wad()`.

**Save path:** Application builds `wad_data` via `create_empty_wad()` ΓåÆ `append_data_to_wad()` ΓåÆ `write_wad()` to file ΓåÆ `calculate_and_store_wadfile_checksum()`.

**Network path:** `get_flat_data()` encapsulates for transfer ΓåÆ receiver calls `inflate_flat_data()` to reconstruct.

Not frame-based; all operations are file I/O triggered by loading/saving game state or level changes.

## External Dependencies
- **FileHandler.h** ΓÇö `FileSpecifier`, `OpenedFile` file abstraction classes
- **Packing.h** ΓÇö `StreamToValue()`, `ValueToStream()`, endianness-aware binary (de)serialization
- **tags.h** ΓÇö Tag type definitions (e.g., `POLYGON_TAG`, `WadDataType`)
- **crc.h** ΓÇö `calculate_crc_for_opened_file()` for checksum computation
- **game_errors.h** ΓÇö `set_game_error()` for error reporting
- **interface.h** ΓÇö `alert_user()` for fatal memory errors; string resource `strERRORS`
- **cseries.h** ΓÇö Misc macros, platform abstractions, type definitions
