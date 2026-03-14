# Source_Files/Network/network_speex.cpp

## File Purpose
Provides Speex encoder/decoder initialization and resource management for network audio compression in Aleph One. Configures audio preprocessing with automatic gain control (AGC) and denoising for real-time voice communication.

## Core Responsibilities
- Initialize Speex narrowband encoder with fixed quality (3) and complexity (4) settings
- Initialize Speex narrowband decoder with enhancement enabled
- Configure audio preprocessing with denoise and automatic gain control
- Manage lifecycle and cleanup of encoder/decoder state and bitstream objects
- Ensure single initialization via guard conditions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SpeexBits` | struct (opaque) | Bitstream container for encoder/decoder data |
| `SpeexPreprocessState` | struct (opaque) | Audio preprocessing state for AGC and denoise |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gEncoderState` | `void*` | global | Opaque Speex encoder state |
| `gEncoderBits` | `SpeexBits` | global | Bitstream for encoded audio data |
| `gDecoderState` | `void*` | global | Opaque Speex decoder state |
| `gDecoderBits` | `SpeexBits` | global | Bitstream for decoder input |
| `gPreprocessState` | `SpeexPreprocessState*` | global | Audio preprocessor (denoise + AGC) |

## Key Functions / Methods

### init_speex_encoder
- **Signature:** `void init_speex_encoder()`
- **Purpose:** Initialize Speex narrowband encoder with quality/complexity tuned for network gameplay, and configure audio preprocessing.
- **Inputs:** None (uses global constants)
- **Outputs/Return:** None (initializes global state)
- **Side effects:** Allocates `gEncoderState`, `gEncoderBits`, `gPreprocessState`; configures encoder CTLs and preprocessor settings
- **Calls:** `speex_encoder_init()`, `speex_encoder_ctl()`, `speex_bits_init()`, `speex_preprocess_state_init()`, `speex_preprocess_ctl()`
- **Notes:** 
  - Guard: `if (!gEncoderState)` prevents re-initialization
  - Quality=3 ΓåÆ 8000 bps; complexity=4 for demanding network play
  - AGC level: `32768 ├ù 0.7` normalizes 16-bit audio to ~70% of max range
  - Both denoise and AGC enabled at initialization

### destroy_speex_encoder
- **Signature:** `void destroy_speex_encoder()`
- **Purpose:** Release encoder and preprocessor resources.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deallocates `gEncoderState`, `gEncoderBits`, `gPreprocessState`; nullifies pointers
- **Calls:** `speex_encoder_destroy()`, `speex_bits_destroy()`, `speex_preprocess_state_destroy()`
- **Notes:** Null-checks before destruction; idempotent (safe to call multiple times)

### init_speex_decoder
- **Signature:** `void init_speex_decoder()`
- **Purpose:** Initialize Speex narrowband decoder with enhancement for better received audio quality.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates `gDecoderState`, initializes `gDecoderBits`, applies decoder configuration
- **Calls:** `speex_decoder_init()`, `speex_decoder_ctl()`, `speex_bits_init()`
- **Notes:** 
  - Guard: `if (gDecoderState == NULL)` prevents re-initialization
  - Enhancement (`SPEEX_SET_ENH=1`) enabled for higher playback quality
  - Sampling rate matches encoder: `kNetworkAudioSampleRate` (8000 Hz)

### destroy_speex_decoder
- **Signature:** `void destroy_speex_decoder()`
- **Purpose:** Release decoder resources.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deallocates `gDecoderState`, `gDecoderBits`; nullifies pointers
- **Calls:** `speex_decoder_destroy()`, `speex_bits_destroy()`
- **Notes:** Null-checks before destruction; idempotent

## Control Flow Notes
This module is called during game network initialization/shutdown. Init functions are invoked before audio frames are processed; destroy functions during connection teardown. Guards prevent redundant allocation if called multiple times. Encoder preprocessing happens at frame-encoding time; decoder operates on incoming compressed frames.

## External Dependencies
- **Speex library** (defined elsewhere):
  - `speex_encoder_init()`, `speex_decoder_init()`, `speex_nb_mode` (narrowband codec mode)
  - `speex_encoder_ctl()`, `speex_decoder_ctl()` (configuration)
  - `speex_bits_init()`, `speex_bits_destroy()` (bitstream management)
  - `speex_preprocess_state_init()`, `speex_preprocess_ctl()` (audio preprocessing)
- **Game constants:** `kNetworkAudioSampleRate` (8000 Hz) from `network_audio_shared.h`
- **Conditional:** Entire file gated by `SPEEX` and `!DISABLE_NETWORKING` preprocessor guards
