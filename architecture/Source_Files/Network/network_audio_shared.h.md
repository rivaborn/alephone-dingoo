# Source_Files/Network/network_audio_shared.h

## File Purpose
Header-only definitions for network audio encoding shared between microphone and speaker modules in Aleph One. Defines the in-memory audio packet header structure and fixed audio format constants used for network transmission.

## Core Responsibilities
- Define `network_audio_header` struct for transmitted audio data
- Define audio format flags (teammate-only filtering)
- Specify network audio format constants (sample rate, bit depth, mono/stereo)
- Provide extensibility hooks (`mReserved` field) for future format changes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `network_audio_header` | struct | In-memory packet header for network audio, containing reserved field and format flags |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kNetworkAudioIsStereo` | const bool | global | Fixed: mono audio (false) |
| `kNetworkAudioIs16Bit` | const bool | global | Fixed: 16-bit samples (true) |
| `kNetworkAudioIsSigned8Bit` | const bool | global | Fixed: unsigned format (false) |
| `kNetworkAudioSampleRate` | const int | global | Fixed: 8000 Hz sample rate |
| `kNetworkAudioBytesPerFrame` | const int | global | Computed: 2 bytes/frame (16-bit mono) |
| `kNetworkAudioForTeammatesOnlyFlag` | enum (0x01) | global | Flag bit to mark audio as team-only distribution |

## Key Functions / Methods
NoneΓÇöthis is a definition-only header.

## Control Flow Notes
Not part of runtime control flow. Used as a configuration module by:
- Network microphone capture routines (source)
- Network speaker playback routines (sink)

Currently hardcoded to 8 kHz, 16-bit unsigned, mono. The file design allows future format negotiation via the `mReserved` field or additional flags, but **implementation note warns that microphone code currently ignores these settings and targets 11025 Hz, unsigned 8-bit mono** ΓÇö any format change requires updating the capture routines as well.

## External Dependencies
- `config.h` ΓÇö build configuration (provides `DISABLE_NETWORKING` guard)
- `cseries.h` ΓÇö base types (`uint32`, `int`) and SDL compatibility layer
- Conditional compilation: entire file disabled if `DISABLE_NETWORKING` is defined
