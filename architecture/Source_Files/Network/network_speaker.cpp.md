# Source_Files/Network/network_speaker.cpp

## File Purpose
Manages playback of received network audio data for multiplayer games. Handles double-buffered audio output via Mac Sound Manager APIs, maintains a queue of incoming network audio, and implements a state machine to start/stop audio playback based on connection status and data availability.

## Core Responsibilities
- Initialize and tear down network audio playback device (`open_network_speaker`, `close_network_speaker`)
- Queue incoming network audio data from remote players (interrupt-safe)
- Fill double buffers with queued audio or generated static when data unavailable
- Manage speaker state machine (off ΓåÆ turning on ΓåÆ on)
- Monitor connection status and suppress audio if no data arrives within threshold
- Generate pseudo-random static/noise to fill gaps and reduce startup delay
- Coordinate with main thread idle processing to start audio playback safely

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `speaker_definition` | struct | Encapsulates channel pointer, double buffer headers, audio queue, block size, connection threshold/status, and speaker state |
| `SndDoubleBuffer` | struct (external) | Sound Manager structure holding audio frame count, flags, and raw PCM data |
| `SndDoubleBufferHeader` | struct (external) | Sound Manager header specifying sample rate (11025 Hz), channels (mono), sample size (8-bit), and buffer pair pointers |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `speaker` | `struct speaker_definition *` | static | Singleton pointer to active speaker state; NULL when not initialized |
| `doubleback_routine_descriptor` | `SndDoubleBackUPP` | static | Function pointer descriptor for the double buffer completion callback invoked by Sound Manager |

## Key Functions / Methods

### open_network_speaker
- **Signature:** `OSErr open_network_speaker(void)`
- **Purpose:** Initialize network audio playback system; allocate buffers, create sound channel, register callback.
- **Inputs:** None (uses hardcoded constants: 1024-byte blocks, 11025 Hz, 8-bit mono, 2-block connection threshold)
- **Outputs/Return:** `OSErr` (noErr on success)
- **Side effects:** Allocates dynamic memory via `NewPtr`, registers `atexit` handler for cleanup, initializes Speex decoder if enabled, populates global `speaker` and `doubleback_routine_descriptor`
- **Calls (visible):** `NewPtr`, `NewPtrClear`, `SndNewChannel`, `quiet_network_speaker`, `init_speex_decoder` (conditional), `MemError`, `atexit`
- **Notes:** Asserts on bad parameters; on error leaves `speaker` NULL without freeing allocated memory (intentional per comment). Sets speaker state to `_speaker_is_off`.

### close_network_speaker
- **Signature:** `void close_network_speaker(void)`
- **Purpose:** Shutdown and deallocate network audio resources.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Disposes sound channel and all allocated buffers; nulls global `speaker` pointer; calls `destroy_speex_decoder` if enabled
- **Calls (visible):** `SndDisposeChannel`, `DisposePtr`, `destroy_speex_decoder` (conditional)
- **Notes:** Safe to call when `speaker` is NULL; only logs warnings on channel disposal errors.

### queue_network_speaker_data
- **Signature:** `void queue_network_speaker_data(byte *buffer, short count)`
- **Purpose:** Enqueue incoming network audio (or static if buffer is NULL) for playback; handle state transitions and trigger buffer filling. **Can be called at interrupt time.**
- **Inputs:** `buffer` (NULL means generate static; otherwise incoming PCM data), `count` (byte count)
- **Outputs/Return:** None
- **Side effects:** Modifies speaker queue, state, and connection status; calls `fill_network_speaker_buffer`, `reset_network_speaker`, and conditionally `SndPlayDoubleBuffer` (Carbon only)
- **Calls (visible):** `fill_network_speaker_buffer`, `reset_network_speaker`, `quiet_network_speaker`, `SndPlayDoubleBuffer`, `logAnomaly3`
- **Notes:** State transitions on first data arrival (off ΓåÆ turning_on) and resets connection_status counter. Silently drops data if queue full; logs anomaly. Carbon path uses reset instead of quiet at interrupt time to avoid unsafe sound commands.

### network_speaker_idle_proc
- **Signature:** `void network_speaker_idle_proc(void)`
- **Purpose:** Called periodically from main thread to progress state machine; triggers audio playback start when enough data buffered.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** On `_speaker_is_turning_on`: silences any prior audio, fills second buffer, calls `SndPlayDoubleBuffer` to begin playback, transitions to `_speaker_is_on`
- **Calls (visible):** `silence_network_speaker`, `fill_network_speaker_buffer`, `SndPlayDoubleBuffer`, `logAnomaly3`
- **Notes:** Only acts in `_speaker_is_turning_on` state; requires at least `block_size` bytes queued.

### fill_network_speaker_buffer
- **Signature:** `void fill_network_speaker_buffer(SndDoubleBufferPtr doubleBufferPtr)`
- **Purpose:** Fill one double buffer with queued audio data or static; update buffer flags and frame count; advance queue.
- **Inputs:** `doubleBufferPtr` (sound buffer to fill)
- **Outputs/Return:** None (modifies buffer in-place)
- **Side effects:** Moves data from queue to buffer, generates static for missing bytes, updates queue pointers, increments connection_status if no data available, sets buffer flags and frame count based on speaker state
- **Calls (visible):** `BlockMove`, `fill_buffer_with_static`, `vhalt`
- **Notes:** Asserts on invalid queue size or state. If connection_status exceeds threshold, transitions to `_speaker_is_off` and marks buffer as last (dbLastBuffer). Empty buffers (no queued data) increment missed-data counter toward shutdown threshold.

### fill_buffer_with_static
- **Signature:** `static void fill_buffer_with_static(byte *buffer, short count)`
- **Purpose:** Generate pseudo-random static noise to fill audio gaps; reduces audible artifacts and startup delay.
- **Inputs:** `buffer` (destination), `count` (byte count to fill)
- **Outputs/Return:** None (modifies buffer in-place)
- **Side effects:** Updates global `speaker->random_seed`
- **Calls (visible):** None
- **Notes:** Uses simple 16-bit LFSR with polynomial 0xb400; divides output by `kStaticAmplitudeReduction` (2) to lower perceived loudness.

### network_speaker_doubleback_procedure
- **Signature:** `static void network_speaker_doubleback_procedure(SndChannelPtr channel, SndDoubleBufferPtr doubleBufferPtr)`
- **Purpose:** Callback invoked by Sound Manager when it finishes playing a buffer and needs the next one filled.
- **Inputs:** `channel` (sound channel, unused), `doubleBufferPtr` (buffer to fill for next playback)
- **Outputs/Return:** None
- **Side effects:** Calls `fill_network_speaker_buffer` to populate the buffer
- **Calls (visible):** `fill_network_speaker_buffer`
- **Notes:** Runs at (possibly) interrupt time. Platform differences (Carbon vs. old Mac) handled via preprocessor; a5 world restoration commented out (legacy 68K code). Cast to plain C function pointer on Carbon (see header comments).

### silence_network_speaker
- **Signature:** `static void silence_network_speaker(void)`
- **Purpose:** Issue Sound Manager commands to stop playback and clear buffer ready flags.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sends flushCmd and quietCmd to sound channel; clears dbFlags on both double buffers
- **Calls (visible):** `SndDoImmediate`
- **Notes:** Internal helper; asserts on sound command errors.

### reset_network_speaker
- **Signature:** `static void reset_network_speaker(void)`
- **Purpose:** Reset speaker state, queue, and connection counter to initial idle state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets state to `_speaker_is_off`, clears connection_status and queue_size
- **Calls (visible):** None
- **Notes:** Does not deallocate buffers or issue sound commands; safe to call anytime.

### quiet_network_speaker
- **Signature:** `void quiet_network_speaker(void)`
- **Purpose:** Public entry point to silence playback and reset state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `silence_network_speaker` and `reset_network_speaker`
- **Calls (visible):** `silence_network_speaker`, `reset_network_speaker`
- **Notes:** Combined cleanup operation; used on initialization and manual muting.

## Control Flow Notes
**Initialization Phase:** `open_network_speaker()` allocates all resources and sets speaker state to `_speaker_is_off`.

**Runtime/Playback Phase:** 
- Remote player audio arrives asynchronously ΓåÆ `queue_network_speaker_data()` (possibly at interrupt time) appends to queue and transitions state from off ΓåÆ turning_on on first data.
- Main thread periodically calls `network_speaker_idle_proc()` to check if enough data is buffered (ΓëÑ block_size); if so, silences prior audio and calls `SndPlayDoubleBuffer()` to start playback, transitioning to `_speaker_is_on`.
- Once playing, Sound Manager continuously calls the doubleback callback `network_speaker_doubleback_procedure()` to request next buffer fill.
- `fill_network_speaker_buffer()` moves queued data into the buffer; if queue is insufficient, fills remainder with static and increments the missed-data counter.
- If missed-data counter exceeds connection threshold (no new data for ~2 callbacks), speaker transitions back to `_speaker_is_off` and marks final buffer.

**Shutdown Phase:** `close_network_speaker()` disposes all resources.

## External Dependencies
- **Includes:** `stdlib.h`, `macintosh_cseries.h`, `CarbonSndPlayDB.h` (Carbon only), `network_sound.h`, `network_speex.h` (conditional), `Logging.h`
- **External symbols used (defined elsewhere):**
  - Sound Manager types/functions: `SndChannelPtr`, `SndDoubleBufferPtr`, `SndDoubleBufferHeaderPtr`, `SndNewChannel`, `SndDisposeChannel`, `SndDoImmediate`, `SndPlayDoubleBuffer`, `CarbonSndPlayDoubleBuffer` (Carbon), `MySndDoImmediate` (Carbon)
  - Memory management: `NewPtr`, `NewPtrClear`, `DisposePtr`, `MemError`
  - System: `atexit`, `BlockMove`
  - Logging: `logAnomaly3`, `warn`, `vhalt`, `csprintf`
  - Speex codec (conditional): `init_speex_decoder`, `destroy_speex_decoder`
  - Utilities: `MIN` macro
