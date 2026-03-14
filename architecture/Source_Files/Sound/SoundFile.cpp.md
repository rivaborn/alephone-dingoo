# Source_Files/Sound/SoundFile.cpp

## File Purpose
Implements sound file parsing, loading, and management for the Aleph One game engine. Handles System 7 sound headers, sound definitions with multiple permutations, and integration with external audio decoders for custom sounds.

## Core Responsibilities
- Parse and validate System 7 sound header formats (standard and extended variants)
- Load audio data from in-memory buffers or file streams
- Manage hierarchical sound organization (sources ΓåÆ definitions ΓåÆ permutations)
- Support lazy loading of sound permutations
- Integrate external audio files via decoder abstraction
- Provide custom sound slot allocation and management
- Track and report audio properties (channels, bit depth, sample rate, loop points)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| SoundHeader | class | Stores single sound metadata (bit depth, channels, rate, loops) and audio data buffer |
| SoundDefinition | class | Represents a sound with multiple permutations, pitch ranges, flags, playback chance |
| SoundFile | class | Master container managing 2D array of sound definitions (sources ├ù sound codes) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| TryLoadingExternal | function | file-static | Decodes external audio files into SoundHeader; not part of public API |

## Key Functions / Methods

### SoundHeader::UnpackStandardSystem7Header
- **Signature:** `bool UnpackStandardSystem7Header(AIStreamBE &header)`
- **Purpose:** Parse a standard System 7 sound header (22 bytes) from big-endian stream
- **Inputs:** Binary stream positioned at header start
- **Outputs/Return:** `true` if valid; `false` if stream read fails
- **Side effects:** Modifies member flags (sixteen_bit, stereo, signed_8bit) and metadata (length, rate, loop_start, loop_end)
- **Calls:** `header.ignore()`, `operator>>()`
- **Notes:** Sets bytes_per_frame to 1, assumes unsigned 8-bit PCM; header_type byte at offset 20 must be 0x00

### SoundHeader::UnpackExtendedSystem7Header
- **Signature:** `bool UnpackExtendedSystem7Header(AIStreamBE &header)`
- **Purpose:** Parse an extended System 7 sound header (64 bytes) supporting 16-bit, stereo, and signed formats
- **Inputs:** Binary stream positioned at header start
- **Outputs/Return:** `true` if valid; `false` on stream error or unsupported format
- **Side effects:** Sets all audio format flags and calculates bytes_per_frame; logs warning if loop offsets appear to be frame-based rather than byte-based
- **Calls:** `header.ignore()`, `operator>>()`, `logWarning3()`
- **Notes:** Validates format tag ('twos' = signed PCM) and compression ID (-1 = uncompressed); if header_type != 0xfe, skips 22 bytes of unused data

### SoundHeader::Load(const uint8\*)
- **Signature:** `bool Load(const uint8* data)`
- **Purpose:** Dispatch parser for in-memory System 7 sound header based on type byte at offset 20
- **Inputs:** Pointer to audio data buffer
- **Outputs/Return:** `true` if header is valid and parsed; `false` if type byte unrecognized
- **Side effects:** Calls Clear(); sets thisΓåÆdata pointer; invokes appropriate unpacker
- **Calls:** `Clear()`, `UnpackStandardSystem7Header()`, `UnpackExtendedSystem7Header()`
- **Notes:** Type 0x00 ΓåÆ standard, 0xff or 0xfe ΓåÆ extended; delegates actual parsing to helpers

### SoundHeader::Load(OpenedFile&)
- **Signature:** `bool Load(OpenedFile &SoundFile)`
- **Purpose:** Load System 7 sound header and audio data from file stream
- **Inputs:** Open file handle
- **Outputs/Return:** `true` if header and audio loaded successfully; `false` on read failure
- **Side effects:** Allocates internal audio buffer via `Load(length)` overload; reads header then audio data
- **Calls:** `SoundFile.GetPosition()`, `SetPosition()`, `Read()`, header unpacker methods
- **Notes:** Peeks at byte 20 to determine header type before reading full header; allocates buffer based on `length` member set by unpacker

### SoundDefinition::Unpack
- **Signature:** `bool Unpack(OpenedFile &SoundFile)`
- **Purpose:** Deserialize sound definition metadata (64-byte header) from file without loading audio
- **Inputs:** Open file handle positioned at definition start
- **Outputs/Return:** `true` if valid; `false` on read error
- **Side effects:** Populates sound_code, behavior_index, flags, pitch ranges, permutation counts, offsets, last_played; resizes sound_offsets vector
- **Calls:** `SoundFile.Read()`, `AIStreamBE` stream operations
- **Notes:** Reads MAXIMUM_PERMUTATIONS_PER_SOUND offsets but does not load audio; sounds vector not populated here

### SoundDefinition::Load
- **Signature:** `bool Load(OpenedFile &SoundFile, bool LoadPermutations)`
- **Purpose:** Load audio data for one or more sound permutations from file
- **Inputs:** File handle, flag to load all permutations or just one
- **Outputs/Return:** `true` if all requested permutations loaded; `false` on any failure
- **Side effects:** Resizes sounds vector; seeks file and loads each SoundHeader; clears vector on error
- **Calls:** `SoundFile.SetPosition()`, `SoundHeader::Load(OpenedFile&)`, vector operations

### SoundFile::Open
- **Signature:** `bool Open(FileSpecifier& SoundFileSpec)`
- **Purpose:** Open and parse a complete sound resource file (snd2 format)
- **Inputs:** File specification
- **Outputs/Return:** `true` if file is valid snd2 and all definitions unpacked; `false` on error
- **Side effects:** Clears prior state; allocates sound_definitions 2D vector; reads 260-byte file header; validates version, tag, counts; keeps file open (opened_sound_file)
- **Calls:** `FileSpecifier::Open()`, `SoundDefinition::Unpack()`, validation macros
- **Notes:** Normalizes v0 files (sound_count=source_count, source_count=1); validates version Γêê {0,1}, tag='snd2', counts ΓëÑ 0

### SoundFile::Close
- **Signature:** `void Close()`
- **Purpose:** Release all sound definitions and associated resources
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears sound_definitions vector (audio buffers released via RAII)
- **Calls:** `vector::clear()`
- **Notes:** Does not close opened_sound_file (managed by auto_ptr)

### SoundFile::NewCustomSoundDefinition
- **Signature:** `int NewCustomSoundDefinition()`
- **Purpose:** Allocate a new custom sound slot and return its index
- **Inputs:** None
- **Outputs/Return:** Zero-based custom sound index (ΓëÑ real_sound_count)
- **Side effects:** Creates empty SoundDefinition with code=19000+index, appends to all sources, increments sound_count, asserts consistency
- **Calls:** `machine_tick_count()`, vector operations
- **Notes:** Author notes discomfort with index tracking ("this makes me squirm"); assertion verifies push_back succeeded

### AddCustomSoundSlot
- **Signature:** `bool AddCustomSoundSlot(int index, const char* file)`
- **Purpose:** Load external audio file and append as a permutation to a custom sound
- **Inputs:** Custom sound index, file path
- **Outputs/Return:** `true` if file decoded and added to all sources; `false` if not a custom sound or decode fails
- **Side effects:** Decodes external file via `TryLoadingExternal()`, appends SoundHeader to sounds vector (up to MAXIMUM_PERMUTATIONS_PER_SOUND), increments permutation count
- **Calls:** `TryLoadingExternal()`, vector operations
- **Notes:** Skips sources if their permutation count is at limit (5); does not check source_count iteration bounds

### TryLoadingExternal (file-static)
- **Signature:** `static bool TryLoadingExternal(SoundHeader& hdr, const char* path)`
- **Purpose:** Decode external audio file (MP3, Ogg, etc.) into SoundHeader via abstracted decoder
- **Inputs:** Output header reference, file path
- **Outputs/Return:** `true` if decoded successfully; `false` if file not found, decoder unavailable, or decode fails
- **Side effects:** Allocates decoded audio buffer; populates hdr with audio format and rate (as fixed-point FIXED_ONE units)
- **Calls:** `FileSpecifier::SetNameWithPath()`, `Decoder::Get()`, `Decoder::Decode()`, `SoundHeader::Load(int32)`
- **Notes:** Rate stored as `(uint32)(FIXED_ONE * decoder->Rate())`; sets loop_start/loop_end to 0; author comments on "weirdness" in interaction with existing loader

### SoundFile::GetSoundDefinition
- **Signature:** `SoundDefinition* GetSoundDefinition(int source, int sound_index)`
- **Purpose:** Retrieve pointer to a sound definition by source and index
- **Inputs:** Source index, sound index within source
- **Outputs/Return:** Pointer to SoundDefinition, or null if out of bounds
- **Side effects:** None
- **Calls:** None
- **Notes:** Bounds-checked; returns 0 on invalid indices

## Control Flow Notes

**Initialization (frame 0):**
- Engine calls `SoundFile::Open()` to load resource file, unpacks all definitions but does not load audio
- Definitions remain resident; audio loaded on demand via `SoundDefinition::Load()`

**Runtime Custom Sounds:**
- Game can call `NewCustomSoundDefinition()` to reserve a slot, then `AddCustomSoundSlot()` to load permutations
- Custom sounds coexist with built-in definitions in the same 2D vector

**Shutdown:**
- `Close()` or `UnloadCustomSounds()` to clean up

## External Dependencies

- **Logging.h:** `logWarning3()` for diagnostic messages
- **csmisc.h:** `machine_tick_count()` for timestamp
- **Decoder.h:** `Decoder::Get()`, `StreamDecoder` interface for external audio decoding
- **FileHandler.h:** `FileSpecifier`, `OpenedFile` for file I/O
- **AStream.h:** `AIStreamBE` (big-endian binary stream reader, "defined elsewhere")
- **SoundFile.h:** Type definitions and public API
- **Standard Library:** `<vector>`, `<memory>` (auto_ptr), `<boost/shared_array.hpp>`
- **Macros:** `FOUR_CHARS_TO_INT()`, `FIXED_ONE` (defined elsewhere)
