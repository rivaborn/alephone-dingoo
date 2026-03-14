# Source_Files/Network/CarbonSndPlayDB.h

## File Purpose
Header file defining Carbon-compatible sound playback structures and APIs. Provides a double-buffered audio playback interface that mimics the legacy `SndPlayDoubleBuffer` but works with the Carbon runtime environment on macOS.

## Core Responsibilities
- Define data structures for double-buffered audio playback
- Declare callback types for buffer completion notifications
- Export public functions for Carbon sound channel operations
- Provide compatibility layer between legacy Sound Manager API and Carbon

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SndDoubleBuffer | struct | Audio frame buffer with sample count, flags, user metadata, and sample data array |
| SndDoubleBufferHeader | struct | Double buffer configuration: channel count, sample rate, compression, buffer pointers, callback |
| SndDoubleBufferHeader2 | struct | Extended header adding format field (OSType) for codec identification |
| SndDoubleBackProcPtr | typedef | Universal Procedure Pointer callback signature for buffer completion events |
| doubleBackProcT | typedef | Standard C function pointer alternative for buffer callbacks |

## Global / File-Static State
None.

## Key Functions / Methods

### CarbonSndPlayDoubleBuffer
- Signature: `OSErr CarbonSndPlayDoubleBuffer(SndChannelPtr chan, SndDoubleBufferHeaderPtr theParams)`
- Purpose: Initiate double-buffered audio playback on a sound channel
- Inputs: `chan` (sound channel handle), `theParams` (playback configuration)
- Outputs/Return: `OSErr` (error code)
- Calls: Not inferable from header

### MySndDoImmediate
- Signature: `OSErr MySndDoImmediate(SndChannelPtr chan, SndCommand * cmd)`
- Purpose: Execute an immediate sound command on a channel
- Inputs: `chan` (sound channel), `cmd` (sound command)
- Outputs/Return: `OSErr` (error code)
- Calls: Not inferable from header

## Control Flow Notes
Frame-based audio streaming model: client fills buffers, callbacks (via `dbhDoubleBack`) notify when playback consumes a buffer, allowing the next buffer to be queued. The `dbBufferReady` and `dbLastBuffer` flags control buffer state.

## External Dependencies
- Mac Sound Manager types: `SndChannelPtr`, `SndCommand`, `OSErr`, `UnsignedFixed`, `OSType`
- Carbon macros: `CALLBACK_API`, `STACK_UPP_TYPE`, `__BEGIN_DECLS`, `__END_DECLS`
- Conditional: `<sys/cdefs.h>` (CodeWarrior compatibility fallback included)
