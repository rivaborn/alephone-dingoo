# Source_Files/Network/network_microphone_shared.cpp

## File Purpose

Provides platform-independent utility routines for network microphone audio capture, resampling, and transmission in Marathon: Aleph One's multiplayer system. Handles audio format conversion (mono/stereo, 8/16-bit), rate resampling to network standard, and Speex compression for efficient network distribution.

## Core Responsibilities

- Announce and store microphone capture format (sample rate, stereo/mono, bit depth)
- Calculate raw capture buffer sizes per network packet based on format parameters
- Extract audio samples from raw capture buffers with format conversions
- Resample audio to network standard sample rate using fixed-point arithmetic
- Encode resampled audio frames using Speex compression
- Manage two-chunk circular buffer input for capture callbacks
- Distribute encoded packets to network or loopback handler
- Support debug-mode local loopback for testing

## Key Types / Data Structures

None defined in this file.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sSamplesPerSecond | uint32 | static | Capture format sample rate (Hz) |
| sStereo | uint32 | static | Capture format: stereo flag |
| s16Bit | uint32 | static | Capture format: 16-bit flag |
| sNumberOfBytesPerSample | uint32 | static | Bytes per sample (accounts for mono/stereo and bit depth) |
| sCaptureBytesPerPacket | int32 | static | Raw bytes needed to fill one network audio packet |
| rate | _fixed | static | Fixed-point resampling ratio (input rate / network rate) |
| counter | _fixed | static | Fixed-point accumulator for resampling phase |
| sOutgoingPacketBuffer | uint8[...] | static | Static buffer for encoded network packets |

## Key Functions / Methods

### announce_microphone_capture_format
- **Signature:** `bool announce_microphone_capture_format(uint32 inSamplesPerSecond, bool inStereo, bool in16Bit)`
- **Purpose:** Initialize global state with microphone format; calculate resampling and buffer parameters
- **Inputs:** Sample rate, stereo flag, 16-bit flag
- **Outputs/Return:** Always true
- **Side effects:** Updates all static format variables; computes rate, counter, sNumberOfBytesPerSample, sCaptureBytesPerPacket
- **Calls:** None visible
- **Notes:** Must be called once before capture begins; sets up fixed-point resampling ratios

### get_capture_byte_count_per_packet
- **Signature:** `int32 get_capture_byte_count_per_packet()`
- **Purpose:** Retrieve the raw capture byte count required per network audio packet
- **Inputs:** None
- **Outputs/Return:** sCaptureBytesPerPacket
- **Side effects:** None
- **Calls:** None (asserts sSamplesPerSecond > 0)
- **Notes:** Guard against calls before announce_microphone_capture_format()

### getSample (inline template)
- **Signature:** `template<bool stereo, bool sixteenBit> inline int16 getSample(void *data)`
- **Purpose:** Extract and convert a single audio sample from raw capture data
- **Inputs:** Pointer to raw sample; template specializes on stereo/16-bit flags
- **Outputs/Return:** int16 sample in canonical form
- **Side effects:** None
- **Calls:** None
- **Notes:** Stereo averages left/right; 8-bit converts unsigned to signed; 16-bit passthrough for mono; all conversions scale to int16

### copy_and_speex_encode_template
- **Signature:** `template<bool stereo, bool sixteenBit> int32 copy_and_speex_encode_template(uint8* outStorage, void* inStorage, int32 inCount)`
- **Purpose:** Resample input samples to network rate and encode frames with Speex
- **Inputs:** Output buffer, input capture buffer, input byte count; template specializes on format
- **Outputs/Return:** Encoded byte count written to output
- **Side effects:** Updates static counter and storedSamples; interacts with global Speex encoder state (gEncoderState, gEncoderBits)
- **Calls:** getSample, speex_bits_reset, speex_encode_int, speex_bits_write
- **Notes:** Uses linear interpolation during resampling; accumulates 160-sample frames for Speex; frame size hardcoded

### copy_and_speex_encode
- **Signature:** `int32 copy_and_speex_encode(uint8* outStorage, void *inStorage, int32 inCount)`
- **Purpose:** Dispatch to correct template instance based on runtime format state
- **Inputs:** Output buffer, input buffer, byte count
- **Outputs/Return:** Encoded byte count
- **Side effects:** Delegates to one of four template specializations
- **Calls:** copy_and_speex_encode_template (conditional on sStereo and s16Bit)
- **Notes:** Avoids template bloat; centralizes format dispatch logic

### send_audio_data (static)
- **Signature:** `static void send_audio_data(void* inData, short inSize)`
- **Purpose:** Transmit encoded audio packet to network or loopback handler
- **Inputs:** Packet buffer, packet size
- **Outputs/Return:** None
- **Side effects:** Calls NetDistributeInformation or received_network_audio_proc
- **Calls:** NetDistributeInformation (network path) or received_network_audio_proc (loopback path, debug-only)
- **Notes:** Applies _force_unique_teams game option; only defined when SPEEX enabled

### copy_and_send_audio_data (main entry point)
- **Signature:** `int32 copy_and_send_audio_data(uint8* inFirstChunkReadPosition, int32 inFirstChunkBytesRemaining, uint8* inSecondChunkReadPosition, int32 inSecondChunkBytesRemaining, bool inForceSend)`
- **Purpose:** Main pipeline: consume capture data from two-chunk circular buffer, resample/encode/transmit packets
- **Inputs:** Two buffer chunks (handles ring wrap), force-send flag
- **Outputs/Return:** Total input bytes consumed
- **Side effects:** Calls copy_and_speex_encode and send_audio_data; modifies buffer positions/counts during loops
- **Calls:** copy_and_speex_encode, send_audio_data, std::min, netcpy (for header endian conversion)
- **Notes:** Assumes Speex output fits within sOutgoingPacketBuffer (kNetworkAudioSamplesPerPacket * kNetworkAudioBytesPerFrame); validates byte alignment to sample boundaries; processes first chunk exhaustion loop, then second chunk loop; inForceSend triggers final packet even if undersized

## Control Flow Notes

The file implements a **capture ΓåÆ resample ΓåÆ encode ΓåÆ transmit** pipeline typically called from an audio callback or network frame handler:

1. **Setup phase:** `announce_microphone_capture_format()` configures global state
2. **Per-capture cycle:** `copy_and_send_audio_data()` drains circular buffer, resamples samples to network rate via fixed-point interpolation, accumulates Speex frames, and sends complete packets
3. **Packet emission:** Triggered when sufficient raw bytes exist or `inForceSend` is true
4. **Network dispatch:** `send_audio_data()` routes to NetDistributeInformation or loopback handler

## External Dependencies

- **network_data_formats.h** ΓÇö `network_audio_header`, `netcpy()` (endian-aware copy)
- **network_distribution_types.h** ΓÇö `kNewNetworkAudioDistributionTypeID`
- **network_speaker_sdl.h** ΓÇö Speaker interface (headers only, no direct calls here)
- **network_speex.h** ΓÇö `gEncoderState`, `gEncoderBits`, Speex state (conditional SPEEX)
- **preferences.h**, **map.h** ΓÇö `GET_GAME_OPTIONS()`, `_force_unique_teams` flag
- **cseries.h** ΓÇö Platform types, `_fixed`, `FIXED_ONE`, STL
- **algorithm** ΓÇö `std::min`, `std::pair`
- **speex/speex.h** ΓÇö Speex codec (conditional SPEEX): `speex_bits_reset`, `speex_encode_int`, `speex_bits_write`

**External symbols called but not defined here:**
- `NetDistributeInformation()` ΓÇö Network packet distribution (defined elsewhere)
- `received_network_audio_proc()` ΓÇö Debug loopback handler (conditional, defined elsewhere)
- `kNetworkAudioSampleRate`, `kNetworkAudioBytesPerFrame` ΓÇö Constants (likely in network_audio_shared.h)

**Compile guards:** Entire file wrapped in `!DISABLE_NETWORKING`; SPEEX codec sections conditional on `SPEEX` define.
