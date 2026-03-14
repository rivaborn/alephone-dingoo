# Source_Files/Misc/vbl_definitions.h

## File Purpose
Header file defining core data structures and prototypes for the replay recording and playback (VBL) system in Aleph One. Provides types for action queues, recording metadata, and replay state management used by `vbl.c` and `vbl_macintosh.c`.

## Core Responsibilities
- Define the action queue structure for buffering player input during recording
- Define the recording header format containing map/player metadata and checksums
- Manage replay session state (validity, speed, recording/playback flags, cache)
- Provide global replay state access via extern declaration
- Declare timer task installation/removal for frame-rate-independent recording
- Declare input preference synchronization function

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ActionQueue` | struct | Circular buffer for player input actions with read/write indices |
| `recording_header` | struct | Immutable metadata header for recorded replays (length, players, map, version, checksum) |
| `replay_private_data` | struct | Complete replay session state: validity, header, playback speed, cache buffers, resource data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `replay` | `struct replay_private_data` | global (extern) | Current replay/recording session state, accessible across VBL module |

## Key Functions / Methods

### get_player_recording_queue
- Signature: `ActionQueue *get_player_recording_queue(short player_index)`
- Purpose: Retrieve the action queue for a specific player during recording
- Inputs: `player_index` ΓÇö player ID (0 to MAXIMUM_NUMBER_OF_PLAYERS-1)
- Outputs/Return: Pointer to the ActionQueue for that player
- Side effects: None (read-only accessor)
- Notes: Macro in release builds for performance; full function in DEBUG builds for bounds checking

### install_timer_task
- Signature: `timer_task_proc install_timer_task(short tasks_per_second, bool (*func)(void))`
- Purpose: Register a periodic callback timer for recording/playback frame updates
- Inputs: `tasks_per_second` ΓÇö frequency; `func` ΓÇö callback returning bool (continue flag)
- Outputs/Return: Opaque timer task handle for later removal
- Side effects: Registers system-level timer callback
- Calls: (platform-specific, likely `vbl_macintosh.c`)

### remove_timer_task
- Signature: `void remove_timer_task(timer_task_proc proc)`
- Purpose: Unregister and stop a previously installed timer task
- Inputs: `proc` ΓÇö timer task handle from `install_timer_task`
- Side effects: Deregisters timer callback

### set_keys_to_match_preferences
- Signature: `void set_keys_to_match_preferences(void)`
- Purpose: Synchronize input key bindings to user preferences
- Side effects: Updates input system state

## Control Flow Notes
This file defines the replay recording/playback infrastructure. Expected flow: on record, `ActionQueue` buffers are filled via timer callbacks; on playback, queues are drained to replay input. The `recording_header` persists map/player/version info for validation during load. Cache buffers (`fsread_buffer`, `location_in_cache`) suggest streaming from disk or resource archives during playback.

## External Dependencies
- Includes: Standard integer types (`int16`, `uint32`, `int32`, `bool`)
- References (defined elsewhere): `struct player_start_data`, `struct game_data` (merged into headers)
- Implicit dependency: Platform-specific timer implementation (`vbl_macintosh.c`)
