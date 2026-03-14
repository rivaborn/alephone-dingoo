# Source_Files/Network/network_microphone_coreaudio.cpp

## File Purpose
macOS CoreAudio implementation for capturing microphone input and sending audio data over the network. Handles hardware initialization, format configuration, and real-time audio frame buffering for the Aleph One game engine's network microphone feature.

## Core Responsibilities
- Initialize and configure CoreAudio HAL (Hardware Abstraction Layer) for microphone input
- Discover and select the system's default audio input device
- Negotiate audio format (sample rate, channels, bit depth) between device and application
- Implement audio input callback to receive frames from CoreAudio
- Buffer and accumulate audio data, sending when packet thresholds are met
- Manage microphone capture state (start/stop recording)
- Clean up audio unit and allocated memory on shutdown

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| AudioUnit | CoreAudio component type | HAL audio unit for input/output control |
| AudioDeviceID | CoreAudio typedef | Identifier for input device |
| AudioStreamBasicDescription | CoreAudio struct | Describes audio format (sample rate, channels, bit depth) |
| AudioBufferList | CoreAudio struct | Container for one or more audio buffers |
| captureBuffer | std::vector\<uint8\> | Dynamic buffer accumulating audio frames until send threshold |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| fAudioUnit | AudioUnit | static | The audio unit instance managing I/O |
| fInputDeviceID | AudioDeviceID | static | ID of selected input device |
| fOutputFormat | AudioStreamBasicDescription | static | Desired audio format (mono, 16-bit PCM) |
| fDeviceFormat | AudioStreamBasicDescription | static | Actual device format (for reference) |
| fAudioSamples | Uint32 | static | Buffer size in frames |
| fAudioBuffer | AudioBufferList* | static | Allocated buffer receiving frames from AudioUnitRender |
| initialized | bool | static | Tracks successful initialization |
| captureBuffer | std::vector\<uint8\> | static | Accumulator for audio data pending send |
| captureBufferSize | Uint32 | static | Current byte count in captureBuffer |
| mic_active | bool | static | Whether capture is currently running |

## Key Functions / Methods

### audio_input_proc
- **Signature:** `static OSStatus audio_input_proc(void *, AudioUnitRenderActionFlags *, const AudioTimeStamp *, UInt32, UInt32, AudioBufferList *)`
- **Purpose:** CoreAudio callback invoked when audio input frames are ready for processing.
- **Inputs:** CoreAudio frame timing metadata and frame count (`inNumberFrames`); final parameter unused.
- **Outputs/Return:** OSStatus error code.
- **Side effects:** Calls `AudioUnitRender()` to populate `fAudioBuffer`; appends audio to `captureBuffer`; triggers `copy_and_send_audio_data()` when buffer reaches packet size.
- **Calls:** `AudioUnitRender()`, `memcpy()`, `get_capture_byte_count_per_packet()`, `copy_and_send_audio_data()`.
- **Notes:** Hard-coded assumption of 16-bit audio (`inNumberFrames * 2` bytes). No error recovery if send fails.

### open_network_microphone
- **Signature:** `OSErr open_network_microphone()`
- **Purpose:** Initialize CoreAudio HAL, configure audio device and format, allocate buffers, and install input callback.
- **Inputs:** None.
- **Outputs/Return:** OSErr error code (0 = noErr on success, -1 if format rejected).
- **Side effects:** Sets global audio unit, device, and format state; allocates `fAudioBuffer` and `captureBuffer`; initializes Speex encoder if compiled in; sets `initialized = true`.
- **Calls:** `FindNextComponent()`, `OpenAComponent()`, `AudioUnitSetProperty()`, `AudioHardwareGetProperty()`, `AudioDeviceSetProperty()`, `AudioUnitGetProperty()`, `AudioUnitInitialize()`, `malloc()`, `announce_microphone_capture_format()`, `get_capture_byte_count_per_packet()`, `init_speex_encoder()`.
- **Notes:** Tries sample rates 8 kHz, 48 kHz, 44.1 kHz, 22 kHz, 11 kHz in order; uses first that succeeds. Enforces mono, 16-bit signed PCM output regardless of device format. Endianness conditionally set for PowerPC.

### set_network_microphone_state
- **Signature:** `void set_network_microphone_state(bool inActive)`
- **Purpose:** Start or stop live audio capture.
- **Inputs:** `inActive` = true to start, false to stop.
- **Outputs/Return:** None.
- **Side effects:** Calls `AudioOutputUnitStart()` or `AudioOutputUnitStop()`; updates `mic_active` flag.
- **Calls:** `AudioOutputUnitStart()`, `AudioOutputUnitStop()`.
- **Notes:** No-op if `initialized` is false. State change only occurs on transition (not idempotent in implementation terms, but guard prevents re-entry).

### close_network_microphone
- **Signature:** `void close_network_microphone()`
- **Purpose:** Clean up audio resources and tear down the audio unit.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Sets `initialized = false`; frees `fAudioBuffer->mBuffers[0].mData` and `fAudioBuffer`; calls `destroy_speex_encoder()` if compiled in.
- **Calls:** `free()`, `destroy_speex_encoder()`.
- **Notes:** Does not validate pointers before free; relies on init to set fAudioBuffer correctly.

### is_network_microphone_implemented
- **Signature:** `bool is_network_microphone_implemented()`
- **Purpose:** Indicate platform support for network microphone.
- **Inputs:** None.
- **Outputs/Return:** Always returns `true` (CoreAudio implementation exists).
- **Side effects:** None.
- **Notes:** Trivial; used for build-time variant detection.

### network_microphone_idle_proc
- **Signature:** `void network_microphone_idle_proc()`
- **Purpose:** Idle/polling hook for microphone subsystem.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** None.
- **Notes:** Empty; CoreAudio handles I/O asynchronously via callback, no polling needed.

## Control Flow Notes
- **Initialization (one-time):** Game calls `open_network_microphone()` at startup to configure audio device and set up the input callback. Format is announced to the shared network code via `announce_microphone_capture_format()`.
- **Frame-by-frame (real-time):** CoreAudio's audio thread invokes `audio_input_proc()` when frames are available. Data is buffered and sent when reaching packet boundaries (no per-frame overhead).
- **State control:** Game calls `set_network_microphone_state(true/false)` to enable/disable transmission.
- **Shutdown (one-time):** Game calls `close_network_microphone()` to free resources and stop capture.
- Not integrated into explicit update/render loops; runs asynchronously on CoreAudio's I/O thread.

## External Dependencies
- **Frameworks:** Carbon/Carbon.h, AudioUnit/AudioUnit.h (CoreAudio on macOS).
- **Internal headers:** `network_microphone_shared.h` (declares `announce_microphone_capture_format()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()`); `cstypes.h` (provides `uint8`, `Uint32`).
- **Conditional:** Speex encoder init/destroy functions (defined elsewhere, gated by `#ifdef SPEEX`).
- **Standard library:** `<vector>` for `captureBuffer`, `<cstdio>` implicitly for `fprintf()`.
