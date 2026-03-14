# Source_Files/Network/network_speex.h

## File Purpose
Header file declaring global Speex encoder/decoder state and management functions for network audio compression in Aleph One. Provides interface for initializing and cleaning up Speex codec resources used in multiplayer audio streaming.

## Core Responsibilities
- Declare global encoder/decoder state pointers and bit buffers
- Declare audio preprocessing state for Speex
- Declare initialization function for Speex encoder
- Declare cleanup function for Speex encoder
- Declare initialization function for Speex decoder
- Declare cleanup function for Speex decoder

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SpeexBits` | opaque type (from speex/speex.h) | Bit buffer for encoder output / decoder input |
| `SpeexPreprocessState` | opaque struct (from speex/speex_preprocess.h) | Audio preprocessing (noise suppression, AGC) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gEncoderState` | `void*` | global | Opaque Speex encoder instance |
| `gEncoderBits` | `SpeexBits` | global | Bit buffer for encoded audio output |
| `gDecoderState` | `void*` | global | Opaque Speex decoder instance |
| `gDecoderBits` | `SpeexBits` | global | Bit buffer for compressed audio input |
| `gPreprocessState` | `SpeexPreprocessState*` | global | Audio preprocessing state (noise reduction) |

## Key Functions / Methods

### init_speex_encoder
- Signature: `void init_speex_encoder()`
- Purpose: Allocate and initialize Speex encoder state and bit buffer
- Inputs: None
- Outputs/Return: None (populates global `gEncoderState` and `gEncoderBits`)
- Side effects: Allocates memory; modifies global state
- Calls: Not visible in this file (implementation in .c file)
- Notes: Must be called before encoding network audio

### destroy_speex_encoder
- Signature: `void destroy_speex_encoder()`
- Purpose: Deallocate Speex encoder state and bit buffer
- Inputs: None
- Outputs/Return: None
- Side effects: Frees memory; nullifies global state
- Calls: Not visible in this file
- Notes: Cleanup counterpart to `init_speex_encoder()`

### init_speex_decoder
- Signature: `void init_speex_decoder()`
- Purpose: Allocate and initialize Speex decoder state and bit buffer
- Inputs: None
- Outputs/Return: None (populates global `gDecoderState` and `gDecoderBits`)
- Side effects: Allocates memory; modifies global state
- Calls: Not visible in this file
- Notes: Must be called before decoding network audio

### destroy_speex_decoder
- Signature: `void destroy_speex_decoder()`
- Purpose: Deallocate Speex decoder state and bit buffer
- Inputs: None
- Outputs/Return: None
- Side effects: Frees memory; nullifies global state
- Calls: Not visible in this file
- Notes: Cleanup counterpart to `init_speex_decoder()`

## Control Flow Notes
Typical lifecycle:
1. **Startup**: Call `init_speex_encoder()` and `init_speex_decoder()` (likely in network module init)
2. **Runtime**: Global state used by network audio codec (encode on transmit, decode on receive)
3. **Shutdown**: Call `destroy_speex_encoder()` and `destroy_speex_decoder()` during cleanup

Only active if `SPEEX` is defined (conditional compilation guards entire interface).

## External Dependencies
- **`speex/speex.h`** ΓÇô Speex codec library (encoder/decoder/bits API)
- **`speex/speex_preprocess.h`** ΓÇô Speex preprocessing (noise suppression, AGC)
- **`cseries.h`** ΓÇô Aleph One common utilities and types
- **`config.h`** ΓÇô Build configuration; `DISABLE_NETWORKING` and `SPEEX` checks
