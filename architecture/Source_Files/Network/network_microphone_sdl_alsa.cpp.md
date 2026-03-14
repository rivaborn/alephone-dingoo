# Source_Files/Network/network_microphone_sdl_alsa.cpp

## File Purpose
Implements ALSA (Advanced Linux Sound Architecture) microphone audio capture for Aleph One's network voice chat. Configures the audio device, manages async capture callbacks, and pipes captured audio to the network transmission layer with optional Speex compression.

## Core Responsibilities
- Open and configure ALSA PCM capture device with hardware parameters (sample rate, format, channels)
- Configure software parameters (availability threshold, start threshold) for low-latency capture
- Register asynchronous capture callback to process incoming audio data
- Manage microphone state transitions (prepare/start/drop device)
- Integrate optional Speex compression encoder initialization/cleanup
- Provide fallback dummy implementation when ALSA is unavailable

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `snd_pcm_t` | opaque pointer | ALSA handle to the PCM capture device |
| `snd_pcm_hw_params_t` | opaque pointer | ALSA hardware parameters structure |
| `snd_async_handler_t` | opaque pointer | ALSA async callback handler registration |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `capture_handle` | `snd_pcm_t*` | static | Opened PCM device; null when closed |
| `hw_params` | `snd_pcm_hw_params_t*` | static | Hardware parameters; freed after `snd_pcm_hw_params()` |
| `initialized` | `bool` | static | True after successful device setup; prevents operations on closed device |
| `active` | `bool` | static | Declared but unused; appears to be vestigial |
| `bytes_per_frame` | `const int` | static | Constant 2 (16-bit mono = 2 bytes per sample) |
| `frames` | `snd_pcm_uframes_t` | static | Period size in frames; computed from capture byte count |
| `mic_active` | `bool` | static | True when capture is running; controls start/stop transitions |
| `pcm_callback` | `snd_async_handler_t*` | static | Callback handler registered with ALSA |

## Key Functions / Methods

### open_network_microphone
- **Signature:** `OSErr open_network_microphone()`
- **Purpose:** Initialize ALSA PCM device and configure for voice capture. Must be called before any other microphone functions.
- **Inputs:** None
- **Outputs/Return:** `0` on success, `-1` on any ALSA error
- **Side effects:** 
  - Opens default ALSA PCM device for capture
  - Allocates and configures hardware/software parameters
  - Sets `initialized = true`
  - Calls `init_speex_encoder()` if `SPEEX` and `network_preferences->use_speex_encoder` are enabled
- **Calls:** 
  - `snd_pcm_open()`, `snd_pcm_hw_params_malloc()`, `snd_pcm_hw_params_any()`
  - `snd_pcm_hw_params_set_access()`, `snd_pcm_hw_params_set_format()`, `snd_pcm_hw_params_set_rate_near()`
  - `announce_microphone_capture_format()` (validates format with shared network layer)
  - `snd_pcm_hw_params_set_channels()`, `snd_pcm_hw_params_set_period_size_near()`, `snd_pcm_hw_params()`
  - `snd_pcm_sw_params_malloc()`, `snd_pcm_sw_params_current()`, `snd_pcm_sw_params_set_avail_min()`, `snd_pcm_sw_params_set_start_threshold()`
- **Notes:** 
  - Hard-coded to 8 kHz mono, 16-bit PCM (voice-quality compression target)
  - Endianness-aware format selection (S16_LE or S16_BE)
  - Error messages printed to stderr only; no exception handling
  - `hw_params` freed after `snd_pcm_hw_params()` call; not accessible afterward

### close_network_microphone
- **Signature:** `void close_network_microphone()`
- **Purpose:** Clean up ALSA device and codec resources.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** 
  - Sets `initialized = false`
  - Calls `snd_pcm_close()` on capture device
  - Nulls `capture_handle`
  - Calls `destroy_speex_encoder()` if `SPEEX` and `network_preferences->use_speex_encoder` are enabled
- **Calls:** `snd_pcm_close()`, optionally `destroy_speex_encoder()`
- **Notes:** Safe to call multiple times; `initialized` check prevents operations on closed device

### CaptureCallback
- **Signature:** `void CaptureCallback(snd_async_handler_t *)`
- **Purpose:** Async callback invoked by ALSA when audio data is available. Reads captured audio and forwards to network transmission.
- **Inputs:** ALSA callback handler pointer (unused)
- **Outputs/Return:** None
- **Side effects:** 
  - Reads audio frames from PCM device into local stack buffer (16384 bytes)
  - Calls `copy_and_send_audio_data()` with captured data (payload to network layer)
  - May call `snd_pcm_prepare()` on underrun (EPIPE)
- **Calls:** `snd_pcm_avail_update()`, `snd_pcm_readi()`, `snd_pcm_prepare()`, `copy_and_send_audio_data()`
- **Notes:** 
  - Loops while at least one period of data is available
  - Static buffer reused across calls; data must be consumed or copied by receiver
  - EPIPE (buffer underrun) handled gracefully by re-preparing device

### set_network_microphone_state
- **Signature:** `void set_network_microphone_state(bool inActive)`
- **Purpose:** Enable or disable audio capture without closing the device.
- **Inputs:** `inActive` ΓÇö true to start capture, false to stop
- **Outputs/Return:** None
- **Side effects:** 
  - If activating: calls `snd_pcm_prepare()`, registers callback with `snd_async_add_pcm_handler()`, calls `snd_pcm_start()`
  - If deactivating: unregisters callback with `snd_async_del_handler()`, calls `snd_pcm_drop()` to discard pending frames
  - Updates `mic_active` state
- **Calls:** `snd_pcm_prepare()`, `snd_async_add_pcm_handler()`, `snd_pcm_start()`, `snd_async_del_handler()`, `snd_pcm_drop()`
- **Notes:** 
  - No-op if `!initialized`
  - No-op if state is unchanged (already active/inactive)
  - Error messages printed to stderr only

### is_network_microphone_implemented
- **Signature:** `bool is_network_microphone_implemented()`
- **Purpose:** Advertise that microphone support is available.
- **Inputs:** None
- **Outputs/Return:** Always `true` for ALSA build
- **Side effects:** None
- **Notes:** Returns `false` in dummy implementation (fallback when `HAVE_ALSA` undefined)

### network_microphone_idle_proc
- **Signature:** `void network_microphone_idle_proc()`
- **Purpose:** Placeholder for periodic idle-time processing. Currently unused.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** None
- **Notes:** Async callback model eliminates need for polling; function kept for API compatibility

## Control Flow Notes
1. **Initialization phase:** `open_network_microphone()` configures device but does not start capture.
2. **Activation phase:** `set_network_microphone_state(true)` registers the async callback and starts the PCM stream; `CaptureCallback()` is invoked automatically by ALSA as audio arrives.
3. **Deactivation phase:** `set_network_microphone_state(false)` unregisters callback and stops stream without closing device.
4. **Shutdown phase:** `close_network_microphone()` closes device and frees resources.

The file conditionally compiles: if `HAVE_ALSA` is undefined, it includes `network_microphone_sdl_dummy.cpp` instead (stub implementation).

## External Dependencies
- **ALSA library:** `#include <alsa/asoundlib.h>` ΓÇö PCM device I/O and parameter configuration
- **Speex codec (conditional):** `#include "network_speex.h"` ΓÇö optional voice compression; encoder/decoder state globals and init/destroy functions
- **Network layer (shared interface):** `#include "network_microphone_shared.h"` ΓÇö `announce_microphone_capture_format()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()`
- **Preferences:** `#include "preferences.h"` ΓÇö `network_preferences` global for Speex encoder flag
- **Core library:** `#include "cseries.h"` ΓÇö common types and macros
- **Standard I/O:** `fprintf(stderr, ...)` for error logging
