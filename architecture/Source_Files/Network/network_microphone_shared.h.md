# Source_Files/Network/network_microphone_shared.h

## File Purpose
Defines the internal interface for network microphone (netmic) implementations to announce audio capture format, transmit captured audio data packets over the network, and query packet size requirements. Implementation-specific; not intended for general engine code.

## Core Responsibilities
- Define the contract for audio format announcement (sample rate, stereo/mono, bit depth)
- Provide buffering and packetization logic for captured audio transmission
- Calculate optimal packet size based on current capture format
- Guard implementation against incomplete or mismatched format specifications

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### announce_microphone_capture_format
- Signature: `bool announce_microphone_capture_format(uint32 inSamplesPerSecond, bool inStereo, bool in16Bit)`
- Purpose: Register the audio capture parameters with the netmic system. Must be called once before any data transmission, and again if capture format changes.
- Inputs: Sample rate (Hz), stereo flag, 16-bit flag (vs. 8-bit)
- Outputs/Return: `bool` indicating whether the format is supported/usable
- Side effects: Updates internal format state used by `copy_and_send_audio_data()` and `get_capture_byte_count_per_packet()`
- Calls: Not inferable from this file
- Notes: Error to call `copy_and_send_audio_data()` without a prior successful call; establishes the baseline for packet size calculations

### copy_and_send_audio_data
- Signature: `int32 copy_and_send_audio_data(uint8* inFirstChunkReadPosition, int32 inFirstChunkBytesRemaining, uint8* inSecondChunkReadPosition, int32 inSecondChunkBytesRemaining, bool inForceSend)`
- Purpose: Buffer and transmit captured audio to network. Accepts up to two contiguous memory chunks (e.g., from a ring buffer).
- Inputs: Two (optional) read pointers with byte counts; force-send flag
- Outputs/Return: Number of bytes consumed from input buffers
- Side effects: Network I/O (packet transmission); internal buffering state modification
- Calls: Not inferable from this file
- Notes: Only sends if internal buffer reaches threshold *or* `inForceSend=true`; caller must check `get_capture_byte_count_per_packet()` to decide if preprocessing is worthwhile before calling

### get_capture_byte_count_per_packet
- Signature: `int32 get_capture_byte_count_per_packet()`
- Purpose: Query the byte threshold for one network packet under the current capture format.
- Inputs: None
- Outputs/Return: Packet size in bytes for the most recently announced format
- Side effects: None
- Calls: Not inferable from this file
- Notes: Error to call without first announcing a capture format via `announce_microphone_capture_format()`

## Control Flow Notes
Fits into runtime audio capture and transmission. Typical sequence:
1. **Init**: Call `announce_microphone_capture_format()` once when capture device opens
2. **Frame/Update**: Repeatedly call `copy_and_send_audio_data()` with chunks from capture buffer; query `get_capture_byte_count_per_packet()` to optimize preprocessing
3. **Format Change**: Re-announce if capture device parameters change
4. **Shutdown**: Implied by file scope; no explicit shutdown function visible

## External Dependencies
- `config.h` (conditional compilation guard: `DISABLE_NETWORKING`)
- All implementation details ("defined elsewhere") ΓÇö functions likely defined in sibling netmic implementation files
