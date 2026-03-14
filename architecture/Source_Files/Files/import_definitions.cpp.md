# Source_Files/Files/import_definitions.cpp

## File Purpose
Manages loading and unpacking physics definition data from WAD files into the game engine. Handles initialization of physics-related structures (monsters, effects, projectiles, weapons, constants) for both single-player initialization and network multiplayer synchronization.

## Core Responsibilities
- Store and manage the active physics file specification
- Initialize all physics definition subsystems on game startup
- Load physics WAD data from disk or network sources
- Extract and unpack individual physics definition types from WAD containers
- Support network physics model transfer for multiplayer games
- Validate physics data versions and handle errors gracefully

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `wad_data` | struct | In-memory representation of loaded WAD file with tag data arrays |
| `wad_header` | struct | File header containing version, checksum, and directory metadata |
| `FileSpecifier` | class | Platform-independent file reference for physics file location |
| `OpenedFile` | class | Active file handle for WAD reading operations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PhysicsFileSpec` | `FileSpecifier` | static (file) | Current physics WAD file path; set by `set_physics_file()` |

## Key Functions / Methods

### set_physics_file
- **Signature:** `void set_physics_file(FileSpecifier& File)`
- **Purpose:** Store the physics file specification for later use.
- **Inputs:** File reference specifying physics WAD location.
- **Outputs/Return:** None.
- **Side effects:** Modifies static `PhysicsFileSpec`.
- **Calls:** Direct assignment only.
- **Notes:** Simple setter; no validation performed.

### set_to_default_physics_file
- **Signature:** `void set_to_default_physics_file(void)`
- **Purpose:** Reset physics file to the default system location.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Modifies static `PhysicsFileSpec` via `get_default_physics_spec()`.
- **Calls:** `get_default_physics_spec()`
- **Notes:** Used during initialization or to restore defaults; commented-out debug output.

### init_physics_wad_data
- **Signature:** `void init_physics_wad_data()`
- **Purpose:** Initialize all physics definition subsystems before loading WAD data.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Resets all physics, monster, effect, projectile, and weapon definitions.
- **Calls:** `init_monster_definitions()`, `init_effect_definitions()`, `init_projectile_definitions()`, `init_physics_constants()`, `init_weapon_definitions()`
- **Notes:** Must be called before `import_physics_wad_data()` to avoid stale data.

### import_definition_structures
- **Signature:** `void import_definition_structures(void)`
- **Purpose:** Main entry point for loading and importing all physics definitions.
- **Inputs:** None (uses `PhysicsFileSpec`).
- **Outputs/Return:** None.
- **Side effects:** Initializes and populates all physics definitions; may fail silently if WAD file unavailable.
- **Calls:** `init_physics_wad_data()`, `get_physics_wad_data()`, `import_physics_wad_data()`, `free_wad()`
- **Notes:** Gracefully handles missing physics file (no error thrown); called during level load.

### get_network_physics_buffer
- **Signature:** `void *get_network_physics_buffer(int32 *physics_length)`
- **Purpose:** Serialize physics data for network transmission.
- **Inputs:** Pointer to length variable (output).
- **Outputs/Return:** Flattened physics data buffer; `*physics_length` set to buffer size or 0 on failure.
- **Side effects:** Allocates buffer; caller responsible for freeing via WAD system.
- **Calls:** `get_flat_data()`
- **Notes:** Returns null if physics file not found; length is 0 in that case.

### process_network_physics_model
- **Signature:** `void process_network_physics_model(void *data)`
- **Purpose:** Receive and apply physics definitions from remote player in network game.
- **Inputs:** Flattened physics WAD buffer (from network).
- **Outputs/Return:** None.
- **Side effects:** Reinitializes all physics definitions; unpacks remote player's physics model.
- **Calls:** `init_physics_wad_data()`, `inflate_flat_data()`, `import_physics_wad_data()`, `free_wad()`
- **Notes:** Allows networked multiplayer to use consistent physics; handles null data gracefully.

### get_physics_wad_data (static)
- **Signature:** `static struct wad_data *get_physics_wad_data(bool *bungie_physics)`
- **Purpose:** Load and parse physics WAD file from disk.
- **Inputs:** Pointer to bool (output) indicating if data is Bungie vs. custom physics version.
- **Outputs/Return:** Parsed WAD data or null if file/format error.
- **Side effects:** Opens/closes file; sets game error on failure.
- **Calls:** `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`, `set_game_error()`
- **Notes:** Validates version against `BUNGIE_PHYSICS_DATA_VERSION` or `PHYSICS_DATA_VERSION`; resets errors at end (masks I/O issues).

### import_physics_wad_data (static)
- **Signature:** `static void import_physics_wad_data(struct wad_data *wad)`
- **Purpose:** Unpack and apply all physics definition types from WAD data.
- **Inputs:** Parsed WAD data.
- **Outputs/Return:** None.
- **Side effects:** Populates global physics definition arrays; asserts on size mismatches.
- **Calls:** `extract_type_from_wad()`, `unpack_monster_definition()`, `unpack_effect_definition()`, `unpack_projectile_definition()`, `unpack_physics_constants()`, `unpack_weapon_definition()`
- **Notes:** Processes five physics tag types: monsters, effects, projectiles, physics constants, weapons. Uses assertions to validate data alignment and counts.

## Control Flow Notes
**Initialization phase:** `import_definition_structures()` is called early in game startup to populate physics from the default or user-selected physics file. **Network phase:** `process_network_physics_model()` allows multiplayer clients to sync physics during level load. **Extraction pattern:** All unpacking delegates to type-specific functions (defined elsewhere), allowing modular physics definition handling.

## External Dependencies
- **WAD system:** `open_wad_file_for_reading`, `read_wad_header`, `read_indexed_wad_from_file`, `close_wad_file`, `extract_type_from_wad`, `free_wad`, `get_flat_data`, `inflate_flat_data` ΓÇö defined elsewhere (wad.h/wad.c)
- **Physics subsystems:** `init_monster_definitions`, `init_effect_definitions`, `init_projectile_definitions`, `init_physics_constants`, `init_weapon_definitions`, `unpack_monster_definition`, `unpack_effect_definition`, etc. ΓÇö defined elsewhere (monsters.h, effects.h, projectiles.h, weapons.h, physics_models.h)
- **File management:** `get_default_physics_spec` ΓÇö from interface.h / shell.h
- **Error handling:** `set_game_error` ΓÇö from game_errors.h
- **Tag constants:** `MONSTER_PHYSICS_TAG`, `EFFECTS_PHYSICS_TAG`, etc. ΓÇö from tags.h
