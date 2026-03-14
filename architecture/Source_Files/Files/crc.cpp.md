# Source_Files/Files/crc.cpp

## File Purpose
Implements CRC (Cyclic Redundancy Check) checksum generation for files and data buffers using lookup-table-based algorithms. Provides CRC32 and CCITT (16-bit) checksums for data integrity verification across file I/O and in-memory operations.

## Core Responsibilities
- Compute CRC32 checksums for files and memory buffers
- Compute CCITT (CRC16) checksums for memory buffers
- Build and manage a 256-entry lookup table for fast CRC32 computation
- Provide file I/O abstraction integration via `OpenedFile` handles
- Support incremental CRC computation on streamed file data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `crc_table` | static pointer | 256-entry lookup table for CRC32 polynomial computation |
| `crc_ccitt_table` | static array | 256-entry lookup table for CCITT CRC16 computation |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `crc_table` | `uint32*` | static | Dynamically allocated lookup table for CRC32; built on-demand and freed after use |
| `crc_ccitt_table` | `uint16[256]` | static | Precomputed lookup table for CCITT CRC16 (hardcoded at compile time) |

## Key Functions / Methods

### calculate_crc_for_file
- **Signature:** `uint32 calculate_crc_for_file(FileSpecifier& File)`
- **Purpose:** Public entry point to compute CRC32 checksum for a file at a given path
- **Inputs:** `FileSpecifier& File` ΓÇô file path/handle abstraction
- **Outputs/Return:** `uint32` ΓÇô computed CRC32 value (0 on I/O failure)
- **Side effects:** Opens/closes file via `OpenedFile`; allocates/frees `crc_table`
- **Calls:** `File.Open()`, `calculate_crc_for_opened_file()`, `OFile.Close()`

### calculate_crc_for_opened_file
- **Signature:** `uint32 calculate_crc_for_opened_file(OpenedFile& OFile)`
- **Purpose:** Compute CRC32 for an already-opened file; allows reuse of open handles
- **Inputs:** `OpenedFile& OFile` ΓÇô open file handle
- **Outputs/Return:** `uint32` ΓÇô computed CRC32 value (0 on failure)
- **Side effects:** Allocates `crc_table`; reads file in `BUFFER_SIZE` (1024-byte) chunks; frees `crc_table`
- **Calls:** `build_crc_table()`, `calculate_file_crc()`, `free_crc_table()`

### calculate_data_crc
- **Signature:** `uint32 calculate_data_crc(unsigned char *buffer, int32 length)`
- **Purpose:** Compute CRC32 for a memory buffer; used for in-memory data validation
- **Inputs:** `buffer` ΓÇô pointer to data; `length` ΓÇô byte count
- **Outputs/Return:** `uint32` ΓÇô computed CRC32
- **Side effects:** Allocates/frees `crc_table`
- **Calls:** `build_crc_table()`, `calculate_buffer_crc()`, `free_crc_table()`
- **Notes:** XORs CRC with `0xFFFFFFFFL` before and after computation for standard CRC32 finalization

### calculate_data_crc_ccitt
- **Signature:** `uint16 calculate_data_crc_ccitt(unsigned char *data, int32 length)`
- **Purpose:** Compute CCITT CRC16 checksum for a memory buffer
- **Inputs:** `data` ΓÇô pointer to buffer; `length` ΓÇô byte count
- **Outputs/Return:** `uint16` ΓÇô computed CCITT CRC16 value
- **Side effects:** None (uses precomputed table)
- **Calls:** Direct table lookup via `crc_ccitt_table`

### build_crc_table
- **Signature:** `static bool build_crc_table(void)`
- **Purpose:** Dynamically generate the CRC32 lookup table using the polynomial `0xEDB88320L`
- **Inputs:** None
- **Outputs/Return:** `bool` ΓÇô success/failure (true if allocation succeeded)
- **Side effects:** Allocates `crc_table` (256 ├ù `uint32`); sets global pointer
- **Notes:** Uses standard bit-shift CRC algorithm; table is polynomial-based and data-independent

### free_crc_table
- **Signature:** `static void free_crc_table(void)`
- **Purpose:** Deallocate the CRC32 lookup table
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deletes `crc_table`; sets to `NULL`

### calculate_buffer_crc
- **Signature:** `static uint32 calculate_buffer_crc(int32 count, uint32 crc, void *buffer)`
- **Purpose:** Incrementally update CRC32 for a block of bytes; inner loop of CRC computation
- **Inputs:** `count` ΓÇô number of bytes; `crc` ΓÇô current CRC state; `buffer` ΓÇô data pointer
- **Outputs/Return:** `uint32` ΓÇô updated CRC value
- **Side effects:** None (pure computation; depends on global `crc_table`)
- **Calls:** Direct access to `crc_table`
- **Notes:** Processes one byte per iteration; XOR-based reflection of standard CRC32

### calculate_file_crc
- **Signature:** `static uint32 calculate_file_crc(unsigned char *buffer, short buffer_size, OpenedFile& OFile)`
- **Purpose:** Internal worker that reads an entire file and computes its CRC32
- **Inputs:** `buffer` ΓÇô I/O buffer (typically 1024 bytes); `buffer_size` ΓÇô buffer size; `OFile` ΓÇô opened file
- **Outputs/Return:** `uint32` ΓÇô CRC32 value (0 on I/O failure)
- **Side effects:** Reads file sequentially from position 0; restores original file position on exit
- **Calls:** `OFile.GetPosition()`, `OFile.GetLength()`, `OFile.SetPosition()`, `OFile.Read()`, `calculate_buffer_crc()`
- **Notes:** Streams file in `buffer_size`-byte chunks; saves/restores file position for non-destructive I/O

## Control Flow Notes
This is a utility module with no integration into game loop (init/frame/update/render/shutdown). Checksums are typically computed during:
- File validation (e.g., integrity checks for maps, scripts, or resources)
- Data serialization/deserialization (e.g., save game verification)
- Off-path operations (not frame-critical)

Table allocation/deallocation is local to each public function call for simplicity (not cached globally).

## External Dependencies
- **`FileHandler.h`** ΓÇô `FileSpecifier` (file path abstraction), `OpenedFile` (file I/O handle)
- **`cseries.h`** ΓÇô common type definitions (`uint32`, `uint16`, `int32`), macros, standard includes
- **`crc.h`** ΓÇô header declaring public API
