# Extras/physics_patches.cpp

## File Purpose
A standalone command-line utility that compares two physics WAD (game data) files and generates a binary patch file containing only the differences. Used for creating incremental physics updates without redistributing entire physics files.

## Core Responsibilities
- Parse command-line arguments (original WAD, new WAD, output patch filename)
- Load two physics WAD files from disk and validate their versions
- Perform byte-by-byte comparison of definition data across all game types
- Identify contiguous regions of changes (deltas) for each definition type
- Create a new WAD containing only changed data with parent checksum metadata
- Write patch file to disk with versioning information

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `wad_data` | struct | In-memory representation of a WAD file (tag count, data pointers, metadata) |
| `wad_header` | struct | WAD file header (version, checksum, directory offset, parent_checksum for patch lineage) |
| `directory_entry` | struct | WAD file directory entry (offset, length, index for in-place modification) |
| `tag_data` | struct | Single data block within a WAD (tag type, data pointer, length, offset) |
| `FileDesc` | typedef | OS-level file descriptor (from macintosh_cseries.h, Mac classic API) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `monster_definitions` | `byte[1]` | static | Stub array to allow extensions.h inclusion (not actually used here) |
| `projectile_definitions` | `byte[1]` | static | Stub array |
| `effect_definitions` | `byte[1]` | static | Stub array |
| `weapon_definitions` | `byte[1]` | static | Stub array |
| `physics_models` | `byte[1]` | static | Stub array |

## Key Functions / Methods

### main
- Signature: `void main(int argc, char **argv)`
- Purpose: Entry point; orchestrates patch generation from command-line arguments
- Inputs: `argc`, `argv` (expects: program_name original_file new_file patch_output)
- Outputs/Return: Calls `exit(0)` on success, `exit(1)` on error
- Side effects: Creates FileDesc structures via FSMakeFSSpec; creates patch file on disk; writes to stderr on errors
- Calls: `FSMakeFSSpec()`, `strcpy()`, `c2pstr()`, `create_delta_wad_file()`, `fprintf()`
- Notes: Mac OS 9ΓÇôspecific file API; fnfErr (file not found) is acceptable for new patch file

### get_physics_wad_from_file
- Signature: `static struct wad_data *get_physics_wad_from_file(FileDesc *file, unsigned long *checksum)`
- Purpose: Load and validate a physics WAD file from disk
- Inputs: `file` pointer to FileDesc, `checksum` pointer to receive parent checksum (may be NULL)
- Outputs/Return: Pointer to heap-allocated `wad_data` struct; NULL on any error
- Side effects: Opens file, allocates WAD data, closes file; prints errors to stderr
- Calls: `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`, `fprintf()`
- Notes: Returns NULL if data_version Γëá PHYSICS_DATA_VERSION; sets checksum by side effect if provided

### write_patchfile
- Signature: `static boolean write_patchfile(FileDesc *file, struct wad_data *wad, short data_version, unsigned long parent_checksum, unsigned long patch_filetype)`
- Purpose: Write a patch WAD to disk with full header, data, and directory
- Inputs: File descriptor, WAD data to write, data version, parent checksum (links to original), file type code
- Outputs/Return: TRUE on success, FALSE if write fails
- Side effects: Creates/overwrites file; allocates temporary directory entries; updates file checksum; closes file handle
- Calls: `create_wadfile()`, `open_wad_file_for_writing()`, `fill_default_wad_header()`, `write_wad_header()`, `write_wad()`, `write_directorys()`, `calculate_and_store_wadfile_checksum()`, `close_wad_file()`
- Notes: Parent checksum allows patch verification; single WAD in patch file (wad_count=1)

### create_delta_wad_file
- Signature: `static boolean create_delta_wad_file(FileDesc *original, FileDesc *new, FileDesc *delta, unsigned long delta_file_type)`
- Purpose: Core delta generation; compares two physics WADs and creates patch WAD
- Inputs: Three FileDesc pointers, file type code for patch
- Outputs/Return: TRUE if patch created successfully (or files identical), FALSE on load error
- Side effects: Allocates original_wad, new_wad, delta_wad; writes patch file; deallocates all WADs; writes debug output to stderr
- Calls: `get_physics_wad_from_file()`, `create_empty_wad()`, `extract_type_from_wad()`, `append_data_to_wad()`, `write_patchfile()`, `free_wad()`, `fprintf()`
- Notes: Iterates through all NUMBER_OF_DEFINITIONS types; uses byte-by-byte scan to find first and last changed byte per type; skips unchanged types

### level_transition_malloc
- Signature: `void *level_transition_malloc(size_t size)`
- Purpose: Malloc wrapper (stub; may be intended as extensibility hook)
- Inputs: `size` in bytes
- Outputs/Return: Pointer to allocated memory
- Side effects: Allocates from heap
- Calls: `malloc()`
- Notes: Unused in this file; likely required by extensions.h or other included headers

## Control Flow Notes
**Initialization phase:** Stub definition arrays created to satisfy extensions.h requirements.  
**Argument parsing phase:** main() validates argc, converts file paths to Mac FSSpec structs.  
**File I/O phase:** Two WADs loaded and validated (version check); original's checksum captured for patch parent link.  
**Comparison phase:** For each definition type, extract raw data blocks; scan for byte differences; track contiguous change regions.  
**Delta encoding:** Only changed bytes are written to output WAD (plus their offset within the definition).  
**Finalization:** Patch WAD written with parent_checksum; both input WADs freed.

## External Dependencies
- **macintosh_cseries.h** ΓÇö Mac classic file APIs (`FSMakeFSSpec`, `FSSpec`, `FileDesc`, `c2pstr`, `OSErr`)
- **wad.h** ΓÇö WAD file I/O (`open_wad_file_for_reading`, `read_wad_header`, `read_indexed_wad_from_file`, `extract_type_from_wad`, `append_data_to_wad`, `create_empty_wad`, `fill_default_wad_header`, `write_wad_header`, `write_directorys`, `calculate_and_store_wadfile_checksum`)
- **extensions.h** ΓÇö Game engine definitions; defines `definitions` array with type metadata (tag, etc.); provides `NUMBER_OF_DEFINITIONS`
- **map.h, effects.h, projectiles.h, monsters.h, weapons.h, items.h, media.h** ΓÇö Game data headers (included to allow extensions.h to be included)
- **tags.h** ΓÇö Tag type constants (transitively via wad.h)
- **stdio.h** (implicit) ΓÇö `fprintf`, `stderr`
- **string.h** (implicit) ΓÇö `strcpy`
