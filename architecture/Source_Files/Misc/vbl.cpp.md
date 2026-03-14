# Source_Files/Misc/vbl.cpp

## File Purpose
Manages keyboard/mouse/joystick input polling, action flag generation, and game recording/replay functionality. Serves as the primary input controller (likely "Vertical Blank" based on classic timing terminology) for Marathon game engine. Converts hardware input into normalized action flags that are either processed immediately or recorded to/replayed from disk files.

## Core Responsibilities
- Poll keyboard, mouse, and joystick hardware; aggregate into 32-bit action flag bitsets
- Manage periodic input task execution (30 Hz by default) synchronized with game ticks
- Record player actions to replay files with run-length compression and replay from disk
- Handle game-state synchronization between input heartbeat and world tick counters
- Parse and apply key binding configurations (supports three preset layouts + custom XML)
- Process special input behaviors: double-click detection, latched toggles, run/walk interchange, swim/sink interchange
- Manage mouse button emulation of keypresses and joystick axis mapping

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ActionQueue` | struct | Circular buffer (write/read indices, uint32 buffer) holding action flags per player; used for both recording and replay |
| `replay_private_data` | struct | Global state for recording/replay: queues, file handles, playback speed, cached file data, header |
| `recording_header` | struct | Metadata written to replay files: num_players, level_number, map_checksum, version, player starts, game_data |
| `key_definition` | struct | Maps one key to one action_flag; SDL or Mac keycode; includes mask (Mac) for bit-extraction |
| `special_flag_data` | struct | Defines post-processing rules for specific action flags: type (_double_flag/_latched_flag), persistence counter |
| `XML_KeyParser` | class | XML element parser for individual `<key>` tags (index + sdl/mac keycode) |
| `XML_KeyboardParser` | class | XML element parser for `<keyboard set="N">` container; delegates to KeyParser |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `heartbeat_count` | int32 | static | Incremented each input poll; must stay synchronized with `dynamic_world->tick_count` |
| `input_task_active` | bool | static | Enables/disables input polling when controller is active/paused |
| `input_task` | timer_task_proc | static | Handle to installed timer task for input polling |
| `FilmFileSpec` | FileSpecifier | static | Path to current replay/recording file |
| `FilmFile` | OpenedFile | static | Open file handle for active recording or replay |
| `current_key_definitions` | key_definition[NUMBER_OF_STANDARD_KEY_DEFINITIONS] | non-static (used by vbl_macintosh.c) | Runtime key bindings (synchronized with `input_preferences->keycodes`) |
| `replay` | replay_private_data | static | Central recording/replay state: queues, file I/O buffers, playback speed, header, resource cache |
| `KeyParser` | XML_KeyParser | static | Reusable XML parser instance for `<key>` elements |
| `KeyboardParser` | XML_KeyboardParser | static | Reusable XML parser instance for `<keyboard>` elements |
| `tm_func` | timer_func | static | Installed timer callback (holds pointer to `input_controller`) |
| `tm_period`, `tm_last`, `tm_accum` | uint32 | static | Timer task period and accumulator for tick-based execution |

## Key Functions / Methods

### initialize_keyboard_controller
- **Signature:** `void initialize_keyboard_controller(void)`
- **Purpose:** One-time setup: initialize globals, install timer task, allocate recording queues, register exit handler
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates `replay.recording_queues` array (MAXIMUM_NUMBER_OF_PLAYERS ActionQueues); each queue allocates `MAXIMUM_QUEUE_SIZE` uint32 buffers; installs timer task; registers `atexit()` handler to call `remove_input_controller()`
- **Calls:** `install_timer_task()`, `atexit()`, `set_keys_to_match_preferences()`, `new` (allocate arrays), `enter_mouse()`
- **Notes:** Asserts that all three key definition sets (standard, left-handed, powerbook) have the same count

### input_controller
- **Signature:** `bool input_controller(void)`
- **Purpose:** Main periodic input task: either pull flags from recording, process network input, or poll keyboard/mouse/joystick
- **Inputs:** None (uses global state: `input_task_active`, `heartbeat_count`, `dynamic_world->tick_count`, `replay.*`, `game_is_networked`)
- **Outputs/Return:** Always returns `true` to reschedule timer task
- **Side effects:** Increments `heartbeat_count`; modifies replay state if replaying; calls `process_action_flags()` which may record or enqueue
- **Calls:** `pull_flags_from_recording()`, `parse_keymap()`, `process_action_flags()`, `set_game_state()`
- **Notes:** Checks heartbeat/tick divergence against `MAXIMUM_TIME_DIFFERENCE` (15 ticks); replay speed can pause (MINIMUM_REPLAY_SPEED) or fast-forward (MAXIMUM_REPLAY_SPEED); maintains phase counter for slow replay

### parse_keymap
- **Signature:** `uint32 parse_keymap(void)`
- **Purpose:** Poll hardware (keyboard via SDL, mouse, joystick) and return aggregated action flags
- **Inputs:** None (reads hardware state, input_preferences modifiers)
- **Outputs/Return:** uint32 action flag bitset
- **Side effects:** Reads keyboard/mouse/joystick state; updates `special_flag_data[]` persistence counters; may modify flags for run/walk and swim/sink interchange based on player state and modifiers
- **Calls:** `SDL_GetKeyState()` (or chat input keymap), `mouse_buttons_become_keypresses()`, `joystick_buttons_become_keypresses()`, `test_mouse()`, `process_joystick_axes()`, `mask_in_absolute_positioning_information()`, `build_terminal_action_flags()`
- **Notes:** Special flag post-processing handles double-click persistence and latched button holding; handles both relative (yaw/pitch/velocity) and absolute positioning modes; terminal mode builds different flags from keymap

### process_action_flags
- **Signature:** `void process_action_flags(short player_identifier, const uint32 *action_flags, short count)`
- **Purpose:** Route action flags to recording queue (if recording) and to real action queues for engine processing
- **Inputs:** player_identifier (0ΓÇôMAXIMUM_NUMBER_OF_PLAYERS), action_flags (array), count (number of flags)
- **Outputs/Return:** None
- **Side effects:** If `replay.game_is_being_recorded`, writes to circular queue; always enqueues to `GetRealActionQueues()`
- **Calls:** `record_action_flags()`, `GetRealActionQueues()->enqueueActionFlags()`
- **Notes:** Wrapper that decouples recording from enqueueing

### save_recording_queue_chunk
- **Signature:** `void save_recording_queue_chunk(short player_index)`
- **Purpose:** Compress and write up to RECORD_CHUNK_SIZE action flags from queue to disk using run-length encoding
- **Inputs:** player_index (which player's queue to save)
- **Outputs/Return:** None
- **Side effects:** Allocates static buffer (once); reads and advances `queue->read_index`; writes to `FilmFile`; updates `replay.header.length`
- **Calls:** `get_player_recording_queue()`, `get_recording_queue_size()`, `ValueToStream()`, `FilmFile.Write()`
- **Notes:** Format is (int16 run_count, uint32 flag_value) pairs; trailing end-of-recording indicator if chunk is partial; uses `INCREMENT_QUEUE_COUNTER` macro for circular indexing

### pull_flags_from_recording
- **Signature:** `static bool pull_flags_from_recording(short count)`
- **Purpose:** Dequeue `count` action flags from all players' replay queues and enqueue them to real action queues
- **Inputs:** count (flags per player to dequeue)
- **Outputs/Return:** `true` if all players had at least one flag available; `false` otherwise (game may transition to demo)
- **Side effects:** Advances read indices for all player queues; enqueues to real action queues
- **Calls:** `get_recording_queue_size()`, `GetRealActionQueues()->enqueueActionFlags()`
- **Notes:** Guards against underflow; used during replay to feed recorded input back to engine

### read_recording_queue_chunks
- **Signature:** `static void read_recording_queue_chunks(void)`
- **Purpose:** Decompress and buffer run-length-encoded action flags from disk into recording queues
- **Inputs:** None (reads from `FilmFile` or `replay.resource_data`)
- **Outputs/Return:** None
- **Side effects:** Advances queue write indices; reads from file or resource; sets `replay.have_read_last_chunk` flag
- **Calls:** `get_player_recording_queue()`, `vblFSRead()`, `StreamToValue()`, `FilmFile.Read()`, `FilmFile.GetPosition()`, `FilmFile.GetLength()`
- **Notes:** Handles two sources: resource data (in-memory) or disk file with LRU cache; decompresses (int16 run_count, uint32 flag) pairs

### vblFSRead
- **Signature:** `static bool vblFSRead(OpenedFile& File, int32 *count, void *dest, bool& HitEOF)`
- **Purpose:** Read from file with disk cache buffer (to reduce I/O syscalls)
- **Inputs:** File (open file handle), count (requested bytes, modified to actual bytes read), dest (buffer), HitEOF (output flag)
- **Outputs/Return:** `true` if read succeeded; `false` otherwise
- **Side effects:** Manages `replay.fsread_buffer`, `replay.location_in_cache`, `replay.bytes_in_cache`; may perform file reads
- **Calls:** `File.Read()`, `File.GetPosition()`, `File.GetLength()`
- **Notes:** Detects EOF by comparing bytes requested vs. actual; refills cache when depleted; tracks file position to compute actual read count

### setup_for_replay_from_file
- **Signature:** `bool setup_for_replay_from_file(FileSpecifier& File, uint32 map_checksum)`
- **Purpose:** Open replay file, verify map availability, initialize replay state and buffers
- **Inputs:** File (path to .film file), map_checksum (unused/checked against header)
- **Outputs/Return:** `true` if file opened and map found; `false` otherwise
- **Side effects:** Opens `FilmFile`; allocates `replay.fsread_buffer`; initializes cache and replay speed; reads header
- **Calls:** `FilmFileSpec.Open()`, `FilmFile.Read()`, `unpack_recording_header()`, `use_map_file()`, `alert_user()`
- **Notes:** Sets `replay.valid`, `replay.game_is_being_replayed`, `replay.replay_speed=1`

### start_recording
- **Signature:** `void start_recording(void)`
- **Purpose:** Create/truncate recording file, write header, initialize recording state
- **Inputs:** None (uses `FilmFileSpec`, `replay.header`)
- **Outputs/Return:** None
- **Side effects:** Deletes existing file at path; creates new file; opens for writing; writes header; sets `replay.game_is_being_recorded`
- **Calls:** `get_recording_filedesc()`, `FilmFileSpec.Delete()`, `FilmFileSpec.Create()`, `FilmFileSpec.Open()`, `pack_recording_header()`, `FilmFile.Write()`
- **Notes:** Assumes header has been set by `set_recording_header_data()` before this is called

### stop_recording
- **Signature:** `void stop_recording(void)`
- **Purpose:** Flush all queues to disk, rewrite header with final length, close file
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** For each player, calls `save_recording_queue_chunk()`; reseeks to file start and rewrites header; closes `FilmFile`; sets `replay.valid=false`
- **Calls:** `save_recording_queue_chunk()`, `FilmFile.SetPosition()`, `pack_recording_header()`, `FilmFile.Write()`, `FilmFile.GetLength()`, `FilmFile.Close()`
- **Notes:** Asserts that final file length matches `replay.header.length`

### increment_heartbeat_count / sync_heartbeat_count
- **Signature:** `void increment_heartbeat_count(int value)` / `void sync_heartbeat_count(void)`
- **Purpose:** Adjust or resynchronize input tick counter to world tick counter
- **Inputs:** `increment_heartbeat_count` takes delta value
- **Outputs/Return:** None
- **Side effects:** Modifies `heartbeat_count`
- **Calls:** None
- **Notes:** `sync_heartbeat_count()` used after level transitions or resets to prevent stalls

## Control Flow Notes

**Initialization:** Game startup calls `initialize_keyboard_controller()` which installs timer task and registers exit handler.

**Per-Frame Loop:**
1. Game's main event loop calls `execute_timer_tasks(time)` (or timer hardware fires)
2. Timer accumulator triggers `input_controller()` at ~30 Hz
3. If replaying, `pull_flags_from_recording()` dequeues flags
4. If keyboard input, `parse_keymap()` polls hardware ΓåÆ `process_action_flags()`
5. `process_action_flags()` enqueues to real action queue and/or records to disk
6. Engine consumes from real action queue each tick

**Recording:** `start_recording()` ΓåÆ per-input `process_action_flags()` ΓåÆ periodic `check_recording_replaying()` flushes chunks ΓåÆ `stop_recording()` finalizes

**Replay:** `setup_for_replay_from_file()` ΓåÆ `input_controller()` calls `pull_flags_from_recording()` ΓåÆ game consumes enqueued flags

**Shutdown:** Exit handler `remove_input_controller()` stops timer task, flushes recording, closes files

## External Dependencies

- **Notable includes:** cseries.h, map.h, interface.h, shell.h, preferences.h, mouse.h, player.h, key_definitions.h, ISp_Support.h, FileHandler.h, ActionQueues.h, Console.h, joystick.h
- **Defined elsewhere:**
  - `dynamic_world` (world state: tick_count, player_count)
  - `input_preferences` (keycodes, modifiers, input_device)
  - `local_player_index`, `local_player` (current player)
  - `game_is_networked` (network game flag)
  - `GetRealActionQueues()` (action queue system)
  - `install_timer_task()`, `remove_timer_task()` (timer management)
  - `set_game_state()`, `get_game_state()` (game flow control)
  - `use_map_file()`, `get_recording_filedesc()` (file/resource management)
  - `StreamToValue()`, `ValueToStream()`, `StreamToBytes()`, etc. (binary packing)
  - SDL functions: `SDL_GetKeyState()`, `SDL_GetTicks()` (if SDL platform)
  - Mouse/joystick routines: `test_mouse()`, `process_joystick_axes()`, `enter_mouse()`, etc.
