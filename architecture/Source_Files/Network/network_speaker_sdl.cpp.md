# Source_Files/Network/network_speaker_sdl.cpp

## File Purpose

Implements real-time network audio playback for SDL platforms in Marathon: Aleph One. Manages queues of sound buffers received from the network, provides padding noise during gaps, and interfaces with the Mixer to feed audio data to the output system. Centralizes memory management to ensure all allocations/deallocations occur on the main thread.

## Core Responsibilities

- Initialize and shut down network speaker resources (buffers, queues, noise data)
- Queue incoming network audio packets for playback
- Manage a circular queue of reusable audio storage buffers
- Provide synthetic noise buffers during playback underruns
- Track consecutive empty dequeues to detect when playback should stop
- Interface with the audio Mixer to provide/retrieve buffer descriptors
- Handle optional Speex codec initialization (if compiled with `SPEEX`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `NetworkSpeakerSoundBufferDescriptor` | struct | Audio buffer with data pointer, length, and flags (defined in network_speaker_sdl.h) |
| `CircularQueue<T>` | template class | FIFO queue for managing sound buffers and reusable data storage |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sSoundBuffers` | `CircularQueue<NetworkSpeakerSoundBufferDescriptor>` | static | Queue of buffer descriptors to send to audio system |
| `sSoundDataBuffers` | `CircularQueue<byte*>` | static | Pool of reusable audio storage buffers returned from Mixer |
| `sNoiseBufferStorage` | `byte*` | static | Pre-allocated noise data (~1/9 second at 11025 Hz) |
| `sNoiseBufferDesc` | `NetworkSpeakerSoundBufferDescriptor` | static | Descriptor wrapping noise buffer |
| `sDryDequeues` | `int` | static | Counter tracking consecutive empty dequeues before stopping |
| `sSpeakerIsOn` | `bool` | static | Flag indicating if speaker is actively playing |

## Key Functions / Methods

### open_network_speaker()
- **Signature:** `OSErr open_network_speaker()`
- **Purpose:** Initialize the network speaker system, allocating all required buffers and resetting state.
- **Inputs:** None
- **Outputs/Return:** `OSErr` (0 on success)
- **Side effects:** Allocates `kNoiseBufferSize` bytes for noise, allocates `kNumSoundDataBuffers` audio storage buffers, resets circular queues, initializes Speex decoder (if `SPEEX` enabled)
- **Calls:** `local_random()` (from world.h), `sSoundBuffers.reset()`, `sSoundDataBuffers.reset()`, `init_speex_decoder()`
- **Notes:** Allocates noise buffer only once (checked via `NULL` pointer). Fills noise data with random values divided by 4.

### queue_network_speaker_data()
- **Signature:** `void queue_network_speaker_data(byte* inData, short inLength)`
- **Purpose:** Queue a chunk of incoming network audio data for playback.
- **Inputs:** `inData` - audio payload bytes; `inLength` - number of bytes
- **Outputs/Return:** None
- **Side effects:** Dequeues a buffer from `sSoundDataBuffers`, copies `inData` into it, enqueues descriptor to `sSoundBuffers`. On first data after idle, primes queue with `kNumPumpPrimes` noise buffers. Sets `sSpeakerIsOn = true`.
- **Calls:** `memcpy()`, `sSoundDataBuffers.peek()`, `sSoundDataBuffers.dequeue()`, `sSoundBuffers.enqueue()`
- **Notes:** Silently discards audio if no free buffers available (prints warning via `fdprintf`). Marks buffers as disposable so Mixer knows to call `release_network_speaker_buffer()`.

### network_speaker_idle_proc()
- **Signature:** `void network_speaker_idle_proc()`
- **Purpose:** Periodic maintenance callback invoked between game updates.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Mixer::instance()->EnsureNetworkAudioPlaying()` if speaker is active
- **Calls:** `Mixer::instance()`, `EnsureNetworkAudioPlaying()`
- **Notes:** Ensures audio playback stays active; implementation details in Mixer.

### dequeue_network_speaker_data()
- **Signature:** `NetworkSpeakerSoundBufferDescriptor* dequeue_network_speaker_data()`
- **Purpose:** Retrieve the next audio buffer for playback (called by audio system during rendering).
- **Inputs:** None
- **Outputs/Return:** Pointer to a `NetworkSpeakerSoundBufferDescriptor`, or `NULL` if underrun threshold exceeded
- **Side effects:** Resets `sDryDequeues` when data available. Increments `sDryDequeues` on underrun. Sets `sSpeakerIsOn = false` and returns `NULL` if `sDryDequeues > kMaxDryDequeues`.
- **Calls:** `sSoundBuffers.getCountOfElements()`, `sSoundBuffers.peek()`, `sSoundBuffers.dequeue()`
- **Notes:** Returns pointer to static `sBufferDesc` which is overwritten on each callΓÇöcaller must not assume pointer validity after next call. Returns noise buffer descriptor when no data available but underrun threshold not yet exceeded.

### close_network_speaker()
- **Signature:** `void close_network_speaker()`
- **Purpose:** Shut down the network speaker system, releasing all allocated resources.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Mixer::instance()->StopNetworkAudio()`, drains remaining buffers from queue (calling `release_network_speaker_buffer()` on disposable ones), deallocates all storage buffers, deallocates noise buffer, destroys Speex decoder (if enabled), resets state variables.
- **Calls:** `Mixer::instance()`, `StopNetworkAudio()`, `dequeue_network_speaker_data()`, `is_sound_data_disposable()`, `release_network_speaker_buffer()`, `destroy_speex_decoder()`
- **Notes:** Safe to call multiple times (checked via `NULL` pointers). Drains queue before deallocating to avoid resource leaks.

### release_network_speaker_buffer()
- **Signature:** `void release_network_speaker_buffer(byte* inBuffer)`
- **Purpose:** Return a used audio storage buffer to the free pool for reuse.
- **Inputs:** `inBuffer` - pointer to buffer previously obtained via `queue_network_speaker_data()`
- **Outputs/Return:** None
- **Side effects:** Enqueues buffer pointer into `sSoundDataBuffers`
- **Calls:** `sSoundDataBuffers.enqueue()`
- **Notes:** Called by Mixer after audio playback completes. Implements the "return queue" design mentioned in file header.

## Control Flow Notes

This module integrates into the frame loop as follows:

1. **Init:** `open_network_speaker()` called once at startup
2. **Per-frame:** `network_speaker_idle_proc()` called periodically to signal Mixer
3. **Audio callback:** Mixer's audio thread calls `dequeue_network_speaker_data()` to fetch buffers during sample rendering
4. **Main thread:** `queue_network_speaker_data()` called when network packets arrive; `release_network_speaker_buffer()` called after Mixer finishes a buffer
5. **Shutdown:** `close_network_speaker()` called at engine shutdown

The design isolates allocation/deallocation to the main thread while allowing the audio callback thread to consume/produce buffer pointers safely via circular queues.

## External Dependencies

- **Includes:**
  - `network_sound.h` ΓÇö interface declarations (public API)
  - `network_speaker_sdl.h` ΓÇö `NetworkSpeakerSoundBufferDescriptor`, `is_sound_data_disposable()`
  - `CircularQueue.h` ΓÇö queue template implementation
  - `world.h` ΓÇö `local_random()`
  - `Mixer.h` ΓÇö `Mixer::instance()`, audio system integration
  - `network_distribution_types.h` ΓÇö distribution type constants (included but unused in this file)
  - `network_speex.h` ΓÇö `init_speex_decoder()`, `destroy_speex_decoder()` (conditional)

- **Defined elsewhere:**
  - `fdprintf()` ΓÇö diagnostic output (likely in cseries/logging)
  - `Mixer` singleton ΓÇö drives audio mixing and playback
  - `local_random()` ΓÇö pseudo-random number generation
