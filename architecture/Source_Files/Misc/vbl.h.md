# Source_Files/Misc/vbl.h

## File Purpose
Header for replay recording/playback system and keyboard input controller. Declares functions for managing game session recordings (capturing player input and state), replaying stored sessions from files or resources, and handling keyboard configuration through XML. Part of the frame update and input processing pipeline.

## Core Responsibilities
- **Replay I/O**: Load replays from files or resource bundles; save recording metadata
- **Recording Management**: Start recording sessions; store/retrieve player count, level, map checksum, version, player spawns, and game state
- **Input Processing**: Handle keyboard controller input; parse keyboard mappings; initialize controller state
- **Heartbeat Timing**: Track frame/timing counters via heartbeat increment
- **Debug Support**: Conditional streaming of recorded flags to debug files (DEBUG_REPLAY mode)
- **XML Configuration**: Expose XML parser interface for keyboard configuration

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class (external) | File/path abstraction; used for replay file handling |
| `recorded_flag` | struct (debug) | Debug entry: flag value + player index; used only when DEBUG_REPLAY is defined |
| `player_start_data` | struct (external) | Player spawn position/state; defined elsewhere |
| `game_data` | struct (external) | Game state snapshot; defined elsewhere |

## Global / File-Static State
None.

## Key Functions / Methods

### setup_for_replay_from_file
- Signature: `bool setup_for_replay_from_file(FileSpecifier& File, uint32 map_checksum)`
- Purpose: Load and initialize a replay session from a file
- Inputs: Replay file specifier; map checksum for validation
- Outputs/Return: `true` if successful, `false` otherwise
- Side effects: I/O; initializes replay playback state
- Calls: (unknown; defined elsewhere in VBL.C)

### setup_replay_from_random_resource
- Signature: `bool setup_replay_from_random_resource(uint32 map_checksum)`
- Purpose: Load a random replay session from bundled resources
- Inputs: Map checksum for validation
- Outputs/Return: `true` if successful, `false` otherwise
- Side effects: I/O; initializes replay playback state
- Calls: (unknown; defined elsewhere)

### start_recording
- Signature: `void start_recording(void)`
- Purpose: Begin a new replay recording session
- Inputs: None (uses current game state)
- Outputs/Return: None
- Side effects: Initializes recording buffers; sets recording mode
- Calls: (unknown; defined elsewhere)

### set_recording_header_data
- Signature: `void set_recording_header_data(short number_of_players, short level_number, uint32 map_checksum, short version, struct player_start_data *starts, struct game_data *game_information)`
- Purpose: Store metadata for current recording session
- Inputs: Player count, level ID, map checksum, version, player spawn array, game state snapshot
- Outputs/Return: None
- Side effects: Modifies recording header in memory/file
- Calls: (unknown; defined elsewhere)

### get_recording_header_data
- Signature: `void get_recording_header_data(short *number_of_players, short *level_number, uint32 *map_checksum, short *version, struct player_start_data *starts, struct game_data *game_information)`
- Purpose: Retrieve metadata from active recording or loaded replay
- Inputs: Pointers to output parameters
- Outputs/Return: Fills all pointer parameters with header data
- Side effects: None
- Calls: (unknown; defined elsewhere)

### input_controller
- Signature: `bool input_controller(void)`
- Purpose: Process one frame of keyboard/controller input
- Inputs: None (reads from input device)
- Outputs/Return: `true` if input was available, `false` otherwise
- Side effects: Updates input state; may record input if recording is active
- Calls: (unknown; defined elsewhere)

### increment_heartbeat_count
- Signature: `void increment_heartbeat_count(int value = 1)`
- Purpose: Advance frame/timing counter by specified amount
- Inputs: Increment value (defaults to 1)
- Outputs/Return: None
- Side effects: Modifies global heartbeat counter
- Calls: (unknown; defined elsewhere)

### initialize_keyboard_controller
- Signature: `void initialize_keyboard_controller(void)`
- Purpose: Set up keyboard input hardware/driver (platform-specific)
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes platform keyboard state
- Calls: (implementation in VBL_MACINTOSH.C or platform equivalent)

### Keyboard_GetParser
- Signature: `XML_ElementParser *Keyboard_GetParser()`
- Purpose: Retrieve XML parser for keyboard configuration documents
- Inputs: None
- Outputs/Return: Pointer to XML parser object (allocate/lifetime unknown)
- Side effects: None (likely factory/getter)
- Calls: (unknown; defined elsewhere)

## Control Flow Notes
Functions fit into frame update loop and initialization:
- **Startup**: `initialize_keyboard_controller()` called during engine init
- **Each Frame**: `input_controller()` + `increment_heartbeat_count()` called in main loop input phase
- **Recording Mode**: `start_recording()` + `set_recording_header_data()` once per session; `input_controller()` records each frame
- **Playback Mode**: `setup_for_replay_from_file/resource()` called before main loop; `get_recording_header_data()` used to restore game state; `input_controller()` reads recorded input instead of hardware

Debug replay functions (`open_stream_file()`, `write_flags()`, `close_stream_file()`) only active when `DEBUG_REPLAY` preprocessor flag is set.

## External Dependencies
- **Includes**: `FileHandler.h` (provides `FileSpecifier` class for file abstraction)
- **External Structs** (defined elsewhere): `player_start_data`, `game_data`
- **External Class**: `XML_ElementParser` (XML configuration parser; likely from common XML handling module)
- **Implied Implementation Files**: `VBL.C` (core replay/input logic), `VBL_MACINTOSH.C` (platform-specific keyboard driver)
