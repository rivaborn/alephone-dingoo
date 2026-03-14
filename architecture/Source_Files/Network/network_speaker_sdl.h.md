# Source_Files/Network/network_speaker_sdl.h

## File Purpose
Interface between SDL network speaker receiving code and SDL sound playback code. Defines structures and functions for dequeuing incoming network audio (realtime microphone data) and managing buffer lifecycle on SDL platforms.

## Core Responsibilities
- Define the `NetworkSpeakerSoundBufferDescriptor` structure for passing audio data between network and sound subsystems
- Provide flag-based metadata (disposability) for buffer management
- Dequeue incoming network audio data for playback
- Return used buffers to the free queue for reuse

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NetworkSpeakerSoundBufferDescriptor` | struct | Container for network audio buffer: byte pointer, length, and flags indicating if data should be released after playback |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kSoundDataIsDisposable` | enum constant (0x01) | file-static | Flag bit indicating the buffer's data pointer must be freed by the dequeuer |

## Key Functions / Methods

### dequeue_network_speaker_data
- Signature: `NetworkSpeakerSoundBufferDescriptor* dequeue_network_speaker_data()`
- Purpose: Retrieve the next queued network audio buffer for sound playback
- Inputs: None
- Outputs/Return: Pointer to the next `NetworkSpeakerSoundBufferDescriptor`, or NULL if queue empty (not specified but implied)
- Side effects: Invalidates the pointer returned by the previous call; dequeues data from internal queue
- Calls: (Not visible in this file; defined elsewhere)
- Notes: Called by sound playback routines and by main thread during shutdown (`close_network_speaker()`); must check `kSoundDataIsDisposable` flag to determine cleanup responsibility

### release_network_speaker_buffer
- Signature: `void release_network_speaker_buffer(byte* inBuffer)`
- Purpose: Return an audio buffer to the free pool after playback
- Inputs: `inBuffer` ΓÇö pointer to buffer data previously returned by `dequeue_network_speaker_data()`
- Outputs/Return: None
- Side effects: Deallocates or recycles buffer; returns it to the freequeue
- Calls: (Not visible in this file; defined elsewhere)
- Notes: Only called if buffer's `kSoundDataIsDisposable` flag is set

### is_sound_data_disposable
- Signature: `__inline__ bool is_sound_data_disposable(NetworkSpeakerSoundBufferDescriptor* inBuffer)`
- Purpose: Utility to safely check if a buffer's data requires explicit cleanup
- Inputs: `inBuffer` ΓÇö pointer to descriptor
- Outputs/Return: `true` if `kSoundDataIsDisposable` bit is set; `false` otherwise
- Side effects: None
- Calls: Bitwise AND against `inBuffer->mFlags`
- Notes: Inline function; simplifies flag checking for callers

## Control Flow Notes
This is a passive interface layer. The sound subsystem drives the control flow: it calls `dequeue_network_speaker_data()` during frame/render updates to fetch queued network audio, optionally calls `release_network_speaker_buffer()` if the disposable flag is set, and repeats. Network receiving code (not in this file) populates the queue asynchronously.

## External Dependencies
- `#include "config.h"` ΓÇö conditional compilation guard (DISABLE_NETWORKING)
- `#include "cseries.h"` ΓÇö provides base types (`byte`, `uint32`), macros, and SDL integration
- Implied: threading/synchronization primitives for queue management (not visible; defined elsewhere)
