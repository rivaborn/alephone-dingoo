# Source_Files/Files/wad_prefs.h

## File Purpose
Defines the public interface for preferences file management in the Aleph One game engine. Provides functions to open, read, and write preference data stored in WAD (Where's All the Data) format, with extensible callback-based initialization and validation. Mac-only UI code for preferences dialogs is also declared.

## Core Responsibilities
- Open and manage preferences files using FileHandler abstraction
- Read preference data blocks by type tag with size validation
- Write modified preferences back to disk
- Support custom initialization callbacks for allocation
- Support custom validation callbacks for data integrity
- (Mac) Declare dialog-based preferences UI with item-hit and teardown handlers
- Prevent preference data duplication via callback mechanism

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `preferences_info` | struct | Internal state: holds FileSpecifier and loaded WAD pointer |
| `preferences_dialog_data` | struct (Mac-only) | Describes a preferences dialog panel (resource ID, callbacks for setup/item-hit/teardown) |

## Global / File-Static State
None.

## Key Functions / Methods

### w_open_preferences_file
- **Signature:** `bool w_open_preferences_file(char *PrefName, Typecode Type);`
- **Purpose:** Open or create a preferences file and allocate internal structures
- **Inputs:** `PrefName` (file name), `Type` (typecode for file creation)
- **Outputs/Return:** `true` if successful, `false` otherwise
- **Side effects:** Allocates/initializes global preferences state; performs file I/O
- **Calls:** (not visible in header; likely calls FileSpecifier::Open and loads WAD)
- **Notes:** Sets up the `preferences_info` structure for later reads/writes

### w_get_data_from_preferences
- **Signature:** `void *w_get_data_from_preferences(WadDataType tag, size_t expected_size, prefs_initializer initialize, prefs_validater validate);`
- **Purpose:** Retrieve a preference data block by tag; allocate, initialize, and validate if needed
- **Inputs:** 
  - `tag`: WadDataType identifier for the preference block
  - `expected_size`: expected size to validate against
  - `initialize`: callback to init newly allocated data (may be NULL)
  - `validate`: callback to verify/repair data, return true if valid/fixed (may be NULL)
- **Outputs/Return:** Pointer to preference data (void*), or NULL if not found/invalid
- **Side effects:** May allocate memory; calls callback functions; modifies WAD if validate fixes data
- **Calls:** (not visible; likely reads from internal WAD, calls callbacks)
- **Notes:** Designed to avoid duplication ΓÇö `initialize` callback usually wraps get_data_from_preferences for other preference types

### w_write_preferences_file
- **Signature:** `void w_write_preferences_file(void);`
- **Purpose:** Flush all modified preferences to disk
- **Inputs:** None (operates on global preferences state)
- **Outputs/Return:** void
- **Side effects:** File I/O; writes WAD to disk
- **Calls:** (not visible; likely uses FileHandler::Open and Write)
- **Notes:** Called after preference changes; no return value indicates fire-and-forget design

### set_preferences (Mac-only)
- **Signature:** `bool set_preferences(struct preferences_dialog_data *funcs, short count, void (*reload_function)(void));`
- **Purpose:** (Mac) Launch a multi-panel preferences dialog and apply changes
- **Inputs:** 
  - `funcs`: array of preference dialog descriptors
  - `count`: number of panels
  - `reload_function`: callback to reload preferences after dialog closes
- **Outputs/Return:** `true` if successful
- **Side effects:** Shows modal dialog; calls setup/item_hit/teardown callbacks; may modify preferences
- **Calls:** Dialog manager (Carbon/Toolbox); callbacks in `preferences_dialog_data`
- **Notes:** Conditional compilation (`#if defined(mac)`); likely wraps Mac Toolbox dialog APIs

## Control Flow Notes
- **Initialization:** `w_open_preferences_file()` must be called once at startup to open the prefs file
- **Per-preference access:** Callers use `w_get_data_from_preferences()` to retrieve/initialize typed data blocks
- **Shutdown/save:** `w_write_preferences_file()` called before exit to persist changes
- **(Mac UI path):** Optional `set_preferences()` dialog flow for interactive editing; user changes trigger teardown callbacks, which likely call get_data_from_preferences to write back
- The global `preferences_info` struct (not user-visible) maintains state across these calls

## External Dependencies
- **FileHandler.h:** `FileSpecifier`, `OpenedResourceFile`, `OpenedFile` abstractions
- **tags.h:** `Typecode` enum, `WadDataType` enum (referenced but not included in this file)
- **Implicit WAD code:** Functions assume a loaded `wad_data` structure (type defined elsewhere)
- **(Mac-only):** Carbon/Toolbox dialog APIs (`DialogPtr`, `Typecode`, resource DITL/STR# management)
