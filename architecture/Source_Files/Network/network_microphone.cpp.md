# Source_Files/Network/network_microphone.cpp

## File Purpose

Implements network microphone audio capture for Aleph One (Marathon engine). Handles initialization of Mac OS sound-input devices, manages async recording buffers, converts captured audio to network format with optional Speex compression, and provides start/stop control for in-game voice transmission.

## Core Responsibilities

- Detect and initialize sound-input hardware capability via Gestalt
- Open/close Sound Manager recording devices with proper resource cleanup
- Configure device sample rate, channels, and sample size; tolerate partial failures
- Manage async sound recording via SPB (Sound Parameter Block) callbacks
- Continuously capture audio and route to network transmission layer via `copy_and_send_audio_data()`
- Support optional Speex audio compression (ifdef SPEEX)
- Provide state control interface (`set_network_microphone_state()`) to toggle recording on/off
- Handle A5 world register save/restore for proper Mac 68k callback semantics

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `sound_device_settings` | struct | Audio device configuration: channels, sample rate, sample size, compression type |
| `net_microphone_data` | struct | Microphone instance state: device refnum, buffers, completion callback, recording flag |
| `siSampleRateAvailableData` | struct | Available sample rates from device; packing-aware layout (short count, Handle array) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `net_microphone_installed` | bool | static | Flag indicating successful initialization and readiness |
| `net_microphone` | struct net_microphone_data | static | Singleton instance holding device state, buffers, callbacks |
| `game_mic_settings` | struct sound_device_settings | static | Desired configuration: mono, 11025 Hz, 8-bit, no compression |

## Key Functions / Methods

### open_network_microphone
- Signature: `OSErr open_network_microphone(void)`
- Purpose: Initialize microphone device, allocate buffers, register completion callback, and validate capture format with network layer.
- Inputs: None; reads global `game_mic_settings` and device capabilities.
- Outputs/Return: OSErr (noErr on success, noHardware/notEnoughMemoryErr on failure).
- Side effects: Registers atexit handler on first call; allocates `MAC_OPTIMAL_MIC_BUFFER_SIZE` buffer (if SPEEX); initializes Speex encoder; sets `net_microphone_installed` to true.
- Calls: Gestalt(), SPBOpenDevice(), SPBGetDeviceInfo(), SPBSetDeviceInfo(), get_device_settings(), set_device_settings(), closest_supported_sample_rate(), announce_microphone_capture_format(), init_speex_encoder(), NewSICompletionUPP().
- Notes: Tolerates `notEnoughHardwareErr` from device settings (clamps to closest supported rate). Requires network layer to approve format via `announce_microphone_capture_format()`. One-time init guarded by static flag.

### close_network_microphone
- Signature: `void close_network_microphone(void)`
- Purpose: Stop recording and release all microphone resources.
- Inputs: None.
- Outputs/Return: void.
- Side effects: Calls SPBStopRecording() if recording; closes device; disposes completion callback UPP; deallocates buffer; calls destroy_speex_encoder(); clears `net_microphone_installed`.
- Calls: SPBStopRecording(), SPBCloseDevice(), DisposeSICompletionUPP(), destroy_speex_encoder().
- Notes: Safe to call multiple times. Queries `net_microphone.recording` to decide whether to stop.

### set_network_microphone_state
- Signature: `void set_network_microphone_state(bool triggered)`
- Purpose: Enable or disable recording; restart recording if completion callback reports an error.
- Inputs: `triggered` (true = start/continue recording, false = stop).
- Outputs/Return: void.
- Side effects: Calls `start_sound_recording()` if transitioning to triggered; calls SPBStopRecording() if transitioning away; updates `net_microphone.recording`.
- Calls: start_sound_recording(), SPBStopRecording(), dprintf().
- Notes: If triggered and already recording, checks `net_microphone.param_block.error` to detect and log async failures. Primarily called from game event handlers.

### is_network_microphone_implemented
- Signature: `bool is_network_microphone_implemented(void)`
- Purpose: Query whether microphone feature is compiled in (always true for this implementation).
- Inputs: None.
- Outputs/Return: bool (always true).
- Side effects: None.
- Calls: None.
- Notes: Exists as a platform-abstraction layer; no guarantee hardware is available.

### has_sound_input_capability
- Signature: `bool has_sound_input_capability(void)`
- Purpose: Check hardware capability via Gestalt.
- Inputs: None.
- Outputs/Return: bool (true if `gestaltHasSoundInputDevice` bit is set).
- Side effects: None.
- Calls: Gestalt().
- Notes: Tolerates Gestalt errors (returns false on error).

### get_device_settings
- Signature: `static OSErr get_device_settings(long refnum, struct sound_device_settings *settings)`
- Purpose: Query device for current audio configuration.
- Inputs: `refnum` (device reference); `settings` (output struct).
- Outputs/Return: OSErr.
- Side effects: Populates `settings->num_channels`, `sample_rate`, `sample_size`, `compression_type`.
- Calls: SPBGetDeviceInfo() ├ù 4.
- Notes: Stops on first error; sequential error checks.

### set_device_settings
- Signature: `static OSErr set_device_settings(long refnum, struct sound_device_settings *settings)`
- Purpose: Apply audio configuration to device; accumulate `notEnoughHardwareErr` and return it at end if any setting failed.
- Inputs: `refnum` (device reference); `settings` (config to apply).
- Outputs/Return: OSErr (noErr if all succeed; notEnoughHardwareErr if any failed).
- Side effects: Configures device; may leave partial state if later settings fail.
- Calls: SPBSetDeviceInfo() ├ù 4.
- Notes: Designed to tolerate hardware limitations (e.g., mono-only device). Continues applying remaining settings even after partial failure.

### closest_supported_sample_rate
- Signature: `static OSErr closest_supported_sample_rate(long refNum, UnsignedFixed *sampleRate)`
- Purpose: Query available sample rates and clamp/snap input rate to nearest supported.
- Inputs: `refNum` (device); `sampleRate` (desired rate, modified in-place).
- Outputs/Return: OSErr.
- Side effects: Modifies `*sampleRate` to nearest available rate.
- Calls: SPBGetDeviceInfo(), DisposeHandle().
- Notes: Handles two cases: continuous range (clip to bounds) or discrete list (find closest). Uses packing-aware layout to extract struct fields from buffer.

### start_sound_recording
- Signature: `static OSErr start_sound_recording(void)`
- Purpose: Initialize SPB and issue async SPBRecord().
- Inputs: None; reads `net_microphone.{refnum, buffer, completion_proc, device_internal_buffer_size}`.
- Outputs/Return: OSErr from SPBRecord().
- Side effects: Populates `net_microphone.param_block`; initiates async recording.
- Calls: obj_clear(), SPBRecord().
- Notes: Buffer size and sample rate configured in `net_microphone.param_block`. Sets up userLong with A5 on 68k builds.

### sound_recording_completed
- Signature: `static pascal void sound_recording_completed(SPBPtr pb)`
- Purpose: Async callback invoked when recording buffer fills; send audio and restart recording.
- Inputs: `pb` (completion parameter block with recorded data).
- Outputs/Return: void.
- Side effects: Calls `copy_and_send_audio_data()` to transmit captured audio; restarts recording; restores A5 on 68k.
- Calls: set_a5() (68k), copy_and_send_audio_data(), SPBRecord().
- Notes: Called at interrupt time (Mac). A5 world must be saved/restored for re-entrant code access. Respawns recording on noErr; ignores abortErr and other errors gracefully.

## Control Flow Notes

**Initialization phase:** `open_network_microphone()` is called once at startup. It checks hardware capability, opens the device, sets sample-rate and format, allocates a buffer, and registers the completion callback. The atexit handler ensures `close_network_microphone()` is called on program exit.

**Recording phase:** When in-game voice is triggered, `set_network_microphone_state(true)` calls `start_sound_recording()`, which issues an async SPBRecord(). The Sound Manager then repeatedly calls `sound_recording_completed()` as the buffer fills. Each callback sends the audio data via `copy_and_send_audio_data()` and respawns a new recording cycle.

**Stop/shutdown phase:** `set_network_microphone_state(false)` stops recording via SPBStopRecording(). `close_network_microphone()` cleans up and is guaranteed to run at exit.

## External Dependencies

- **Mac OS Sound Manager:** SPBOpenDevice, SPBCloseDevice, SPBGetDeviceInfo, SPBSetDeviceInfo, SPBRecord, SPBStopRecording, NewSICompletionUPP, DisposeSICompletionUPP
- **Gestalt (Mac OS):** gestaltSoundAttr, gestaltHasSoundInputDevice
- **Speex (optional, ifdef SPEEX):** init_speex_encoder(), destroy_speex_encoder()
- **Network protocol layer (network_microphone_shared.h):** announce_microphone_capture_format(), copy_and_send_audio_data(), get_capture_byte_count_per_packet()
- **Utility macros (cseries.h):** obj_clear()
- **Platform abstractions (cseries.h, shell.h, interface.h):** dprintf()
