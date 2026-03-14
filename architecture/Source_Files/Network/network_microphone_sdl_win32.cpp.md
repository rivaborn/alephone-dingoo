# Source_Files/Network/network_microphone_sdl_win32.cpp

## File Purpose
DirectSoundCapture (Win32 DirectX)-based network microphone implementation for Marathon: Aleph One. Captures audio input from the system microphone via a circular buffer and transmits it over the network for multiplayer voice communication.

## Core Responsibilities
- Initialize and configure DirectSoundCapture device with hardware format detection
- Manage circular audio capture buffer and read position tracking
- Select optimal audio format from device-supported options (11 kHzΓÇô44 kHz, mono/stereo, 8-bit/16-bit)
- Continuously capture and transmit audio data in packet-sized chunks during idle processing
- Start/stop audio transmission on demand
- Integrate with optional Speex audio compression codec
- Clean up DirectX resources on shutdown

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CaptureFormat | struct | Audio format descriptor: WAVE_FORMAT identifier, sample rate, stereo flag, 16-bit flag |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sDirectSoundCapture | LPDIRECTSOUNDCAPTURE | static | IDirectSoundCapture interface pointer |
| sCaptureBuffer | LPDIRECTSOUNDCAPTUREBUFFER | static | Circular capture buffer interface |
| sCaptureBufferSize | int | static | Size of circular buffer in bytes |
| sCaptureSystemReady | bool | static | Initialization completion flag |
| sNextReadStartPosition | int | static | Current read position in circular buffer |
| sCaptureFormatIndex | int | static | Index of selected format in preferences array |
| sTransmittingAudio | bool | static | Audio transmission active flag |
| sFormatPreferences | CaptureFormat[] | static | Array of 12 supported format options (11 kHz to 44 kHz) |

## Key Functions / Methods

### open_network_microphone
- Signature: `OSErr open_network_microphone()`
- Purpose: Initialize DirectSoundCapture device, detect supported formats, allocate circular buffer, announce format to shared code
- Inputs: None (uses static preferences array)
- Outputs/Return: OSErr (always returns 0; errors logged, not returned)
- Side effects: Allocates DirectX resources; writes static state variables; calls `announce_microphone_capture_format()`; optionally initializes Speex encoder
- Calls: `DirectSoundCaptureCreate()`, `GetCaps()`, `CreateCaptureBuffer()`, `announce_microphone_capture_format()`, `init_speex_encoder()` (conditional)
- Notes: Performs hardware format discovery by iterating preferences array in order; asserts no double-open; half-second buffer size hardcoded

### transmit_captured_data
- Signature: `static void transmit_captured_data(bool inForceSend)`
- Purpose: Read available data from circular buffer and transmit in packet-sized chunks
- Inputs: `inForceSend` (bool) ΓÇô force transmission of partial packet at stream end
- Outputs/Return: None
- Side effects: Reads from `sCaptureBuffer`; updates `sNextReadStartPosition`; calls `copy_and_send_audio_data()` to process audio
- Calls: `GetCurrentPosition()`, `Lock()`, `Unlock()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()`
- Notes: Handles circular buffer wraparound; buffer split into two chunks when read wraps; conditionally sends only if packet-full or forced

### start_transmitting_audio / stop_transmitting_audio
- Signature: `static void start_transmitting_audio()` / `static void stop_transmitting_audio()`
- Purpose: Start/stop circular buffer recording; flush remaining data on stop
- Inputs: None
- Outputs/Return: None
- Side effects: Updates `sTransmittingAudio` flag; calls `transmit_captured_data(true)` on stop
- Calls: `Start(DSCBSTART_LOOPING)`, `Stop()`, `transmit_captured_data()`
- Notes: Asserts system ready and correct current state

### close_network_microphone
- Signature: `void close_network_microphone()`
- Purpose: Release DirectX resources, stop transmission, clean up optional codec
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `Release()` on capture interfaces; conditionally calls `destroy_speex_encoder()`; zeroes static pointers
- Calls: `stop_transmitting_audio()`, `Release()`, `destroy_speex_encoder()` (conditional)

### set_network_microphone_state
- Signature: `void set_network_microphone_state(bool inActive)`
- Purpose: Enable/disable audio transmission
- Inputs: `inActive` (bool)
- Outputs/Return: None
- Side effects: Calls `start_transmitting_audio()` or `stop_transmitting_audio()`

### network_microphone_idle_proc
- Signature: `void network_microphone_idle_proc()`
- Purpose: Periodic idle hook to transmit captured data
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `transmit_captured_data(false)` if transmission active

### Utility functions
- `is_network_microphone_implemented()` ΓåÆ returns `true`
- `has_sound_input_capability()` ΓåÆ returns `true` (unconditional; comment notes should actually check)

## Control Flow Notes
**Init**: `open_network_microphone()` called at engine startup; discovers hardware format, allocates half-second circular buffer.  
**Runtime**: `network_microphone_idle_proc()` called on idle; reads circular buffer and transmits captured packets.  
**State**: `set_network_microphone_state()` toggles transmission on demand.  
**Shutdown**: `close_network_microphone()` releases DirectX resources.

## External Dependencies
- `<dsound.h>`: DirectSoundCapture API (Windows DirectX)
- `cseries.h`: Engine primitives and types
- `network_microphone_shared.h`: Shared interface functions (`announce_microphone_capture_format`, `copy_and_send_audio_data`, `get_capture_byte_count_per_packet`)
- `network_speaker_sdl.h`: Audio definitions
- `Logging.h`: Logging macros (`logContext`, `logAnomaly`, `logAnomaly1`, `logAnomaly3`)
- `preferences.h`: Global `network_preferences` (Speex encoder flag)
- `network_speex.h`: Optional codec (`init_speex_encoder`, `destroy_speex_encoder`) ΓÇô conditional on `SPEEX` macro
