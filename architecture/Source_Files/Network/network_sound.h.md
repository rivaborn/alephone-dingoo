# Source_Files/Network/network_sound.h

## File Purpose
Main interface header for network audio support in Aleph One (Marathon engine). Declares functions for bidirectional network audio: speaker system for receiving/playing back remote player audio, and microphone system for capturing/transmitting local player audio over the network.

## Core Responsibilities
- Initialize, control, and shut down network speaker (remote audio playback) system
- Queue incoming network audio data for playback and silence the speaker
- Mute individual players' microphones or clear all mutes
- Detect and initialize network microphone (audio capture) capabilities
- Activate/deactivate local microphone and process captured audio for transmission
- Provide idle entry points for periodic audio processing during game loop

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### open_network_speaker
- Signature: `OSErr open_network_speaker();`
- Purpose: Initialize the network speaker system for receiving and playing remote player audio.
- Inputs: None.
- Outputs/Return: `OSErr` (error code; 0 = success).
- Side effects: Allocates/initializes speaker resources (audio buffers, playback state).
- Calls: Not visible in this file.
- Notes: Must be called by main thread before any audio reception; called once during initialization.

### network_speaker_idle_proc
- Signature: `void network_speaker_idle_proc();`
- Purpose: Process queued network audio and manage playback; called periodically during game loop.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Dequeues audio, manages playback hardware, updates internal state.
- Calls: Not visible in this file.
- Notes: Should be called from main game update loop at regular intervals.

### close_network_speaker
- Signature: `void close_network_speaker();`
- Purpose: Shut down network speaker system and release resources.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Frees audio buffers, stops playback.
- Calls: Not visible in this file.

### queue_network_speaker_data
- Signature: `void queue_network_speaker_data(byte* inData, short inLength);`
- Purpose: Queue raw audio data (from received packets or internal sources) for playback.
- Inputs: `inData` (pointer to audio samples), `inLength` (byte count).
- Outputs/Return: None.
- Side effects: Buffers audio for later playback.
- Calls: Not visible in this file.

### received_network_audio_proc
- Signature: `void received_network_audio_proc(void *buffer, short buffer_size, short player_index);`
- Purpose: Callback invoked by network code to deliver incoming audio from a remote player.
- Inputs: `buffer` (audio data), `buffer_size` (byte count), `player_index` (which player).
- Outputs/Return: None.
- Side effects: Stores audio (likely calls `queue_network_speaker_data` internally).
- Calls: Not visible in this file.

### quiet_network_speaker
- Signature: `void quiet_network_speaker(void);`
- Purpose: Silence the speaker (mute all playback).
- Inputs: None.
- Outputs/Return: None.
- Side effects: Stops or mutes audio output.
- Calls: Not visible in this file.

### mute_player_mic / clear_player_mic_mutes
- Signatures: `void mute_player_mic(short player_index);`, `void clear_player_mic_mutes();`
- Purpose: Silence a specific player's outgoing audio, or clear all mutes.
- Inputs: `player_index` (which player to mute).
- Outputs/Return: None.
- Side effects: Marks microphone as muted/unmuted; affects what audio is transmitted.
- Calls: Not visible in this file.

### is_network_microphone_implemented
- Signature: `bool is_network_microphone_implemented();`
- Purpose: Detect whether network microphone system is compiled/available.
- Inputs: None.
- Outputs/Return: `bool` (true = system exists, false = unavailable; does not guarantee hardware).
- Side effects: None.
- Notes: Comment: "false means you have no hope whatsoever."

### has_sound_input_capability
- Signature: `bool has_sound_input_capability(void);`
- Purpose: Check for actual audio input hardware/driver support (more accurate than `is_network_microphone_implemented`).
- Inputs: None.
- Outputs/Return: `bool`.
- Side effects: None.

### open_network_microphone
- Signature: `OSErr open_network_microphone();`
- Purpose: Initialize the network microphone system for audio capture.
- Inputs: None.
- Outputs/Return: `OSErr`.
- Side effects: Opens audio input device, allocates capture buffers.
- Calls: Not visible in this file.
- Notes: Do not call twice without intervening close.

### set_network_microphone_state
- Signature: `void set_network_microphone_state(bool inActive);`
- Purpose: Activate or deactivate an already-open microphone.
- Inputs: `inActive` (true = on, false = off).
- Outputs/Return: None.
- Side effects: Starts/stops audio capture.
- Calls: Not visible in this file.

### network_microphone_idle_proc
- Signature: `void network_microphone_idle_proc();`
- Purpose: Process captured audio periodically; called from main game loop.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Reads capture buffers, encodes/queues audio for transmission.
- Calls: Not visible in this file.

### close_network_microphone
- Signature: `void close_network_microphone();`
- Purpose: Clean up and close the microphone system.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Releases audio input device and buffers.
- Calls: Not visible in this file.
- Notes: Multiple calls are safe.

## Control Flow Notes
Network audio is gated by `DISABLE_NETWORKING` preprocessor constant. The speaker and microphone systems are independent and follow similar lifecycle patterns:

- **Speaker**: `open_network_speaker()` ΓåÆ periodic `network_speaker_idle_proc()` calls (in game loop) ΓåÆ `close_network_speaker()` at shutdown.
- **Microphone**: `open_network_microphone()` ΓåÆ `set_network_microphone_state()` (control) ΓåÆ periodic `network_microphone_idle_proc()` calls (in game loop) ΓåÆ `close_network_microphone()` at shutdown.
- **Integration**: `received_network_audio_proc()` is called by the network layer when audio packets arrive; implementations of speaker and microphone idle procs likely exchange data with the network stack.

## External Dependencies
- **Includes**: `config.h` (configuration constants), `cseries.h` (platform abstractions, types).
- **Defined elsewhere**: 
  - `OSErr` type (platform error code, likely from cseries).
  - `byte` type (likely from cstypes).
  - Implementations: `NETWORK_SPEAKER.C`, `NETWORK_MICROPHONE.C`.
