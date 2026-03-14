# Source_Files/Files/wad_prefs.cpp

## File Purpose
Manages persistent preferences storage using the WAD (Where's All the Data?) file format. Provides functions to initialize, load, retrieve, and persist game preferences to disk with automatic recovery from corruption or missing files.

## Core Responsibilities
- Opens and initializes the preferences WAD file at startup, platform-specific file paths
- Loads preferences from disk into a WAD structure in memory
- Retrieves typed preference data items with optional initialization and validation callbacks
- Writes modified preferences back to disk with atomic file replacement
- Handles file errors gracefully by creating empty preference containers
- Manages the global preferences state and validates preference data integrity

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `preferences_info` | struct | Holds file reference and in-memory WAD structure for preferences |
| `wad_data` | struct (external) | Generic WAD container holding preference entries |
| `FileSpecifier` | class (external) | Platform-abstracted file path and metadata |
| `OpenedFile` | struct (external) | Active file handle for reading/writing WAD data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `prefInfo` | `preferences_info*` | static (file-scoped) | Global preferences container; NULL if uninitialized |

## Key Functions / Methods

### w_open_preferences_file
- **Signature:** `bool w_open_preferences_file(char *PrefName, Typecode Type)`
- **Purpose:** Opens preferences file and initializes global preferences state; creates new file if missing.
- **Inputs:** `PrefName` (filename string), `Type` (file typecode for platform metadata)
- **Outputs/Return:** `true` if preferences initialized successfully, `false` if WAD is invalid
- **Side effects:** Allocates `prefInfo` global; may create new preferences file on disk; clears/sets game errors
- **Calls:** `new`, `SetParentToPreferences`/`SetToPreferencesDir` (platform-specific), `load_preferences`, `error_pending`, `get_game_error`, `set_game_error`, `create_empty_wad`, `w_write_preferences_file`, `Delete`
- **Notes:** Uses try/catch to handle allocation failures; gracefully falls back to empty WAD on corruption; platform-specific file path setup (Mac vs. SDL)

### w_get_data_from_preferences
- **Signature:** `void *w_get_data_from_preferences(WadDataType tag, size_t expected_size, prefs_initializer initialize, prefs_validater validate)`
- **Purpose:** Retrieves a preference item by tag; auto-initializes or validates if missing/invalid.
- **Inputs:** `tag` (preference ID), `expected_size` (expected data length), `initialize` (callback to populate new data), `validate` (callback to check/fix data, returns true if modified)
- **Outputs/Return:** Raw pointer to preference data within WAD (managed by WAD, not caller)
- **Side effects:** May allocate temporary buffers; appends new/corrected data to WAD; persists changes in WAD
- **Calls:** `assert`, `extract_type_from_wad`, `malloc`, `free`, `initialize` (callback), `append_data_to_wad`, `validate` (callback), `memcpy`
- **Notes:** Size mismatch triggers reinitialization; temporary copies avoid in-place modification of WAD pointers; validation happens only if callback provided and returns true

### w_write_preferences_file
- **Signature:** `void w_write_preferences_file(void)`
- **Purpose:** Persists in-memory preferences WAD to disk file.
- **Inputs:** None (uses global `prefInfo`)
- **Outputs/Return:** None
- **Side effects:** Deletes old preferences file; writes new WAD to disk; modifies game error state
- **Calls:** `error_pending`, `set_game_error`, `assert`, `GetType`, `Delete`, `Create`, `open_wad_file_for_writing`, `fill_default_wad_header`, `write_wad_header`, `calculate_wad_length`, `set_indexed_directory_offset_and_length`, `write_wad`, `write_directorys`, `close_wad_file`
- **Notes:** Clears pending errors before write; deletes old file before creation to avoid stale data; writes header twice (before and after directory); asserts on write failures

### load_preferences (static)
- **Signature:** `static void load_preferences(void)`
- **Purpose:** Loads preferences from disk file into memory; called by `w_open_preferences_file` and on reload.
- **Inputs:** None (uses global `prefInfo` for file path)
- **Outputs/Return:** None
- **Side effects:** Frees old WAD if present; allocates new WAD from file; sets game errors on read failure
- **Calls:** `free_wad`, `open_wad_file_for_reading`, `read_wad_header`, `read_indexed_wad_from_file`, `set_game_error`, `close_wad_file`
- **Notes:** Gracefully degrades if WAD version unknown; leaves `prefInfo->wad` as NULL if read fails

## Control Flow Notes
**Initialization phase:** `w_open_preferences_file` is the entry point, called once at startup to load or create the preferences file.
**Runtime:** `w_get_data_from_preferences` is called on-demand to fetch preference items; initializes or validates them lazily.
**Persistence:** `w_write_preferences_file` is called at shutdown or on preference changes to save to disk.
**Reload:** `load_preferences` can be invoked internally to refresh from disk if needed.

## External Dependencies
- **Notable includes:** `cseries.h` (platform abstractions), `wad.h` (WAD format and I/O), `game_errors.h` (error codes), `FileHandler.h` (FileSpecifier/OpenedFile classes, inferred)
- **External symbols:** `extract_type_from_wad`, `append_data_to_wad`, `create_empty_wad`, `free_wad`, `read_indexed_wad_from_file`, `calculate_wad_length`, `fill_default_wad_header`, `write_wad_header`, `write_directorys`, `write_wad`, `open_wad_file_for_reading`, `open_wad_file_for_writing`, `close_wad_file`, `read_wad_header`, `set_indexed_directory_offset_and_length` (from wad module); `set_game_error`, `get_game_error`, `error_pending` (from game_errors module); `FileSpecifier`, `OpenedFile` platform file handlers
