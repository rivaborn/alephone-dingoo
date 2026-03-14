# Source_Files/Sound/ReplacementSounds.cpp

## File Purpose
Implements sound replacement management for the Aleph One engine, allowing external audio files to override in-game sounds. Manages a singleton registry of sound options indexed by sound ID and slot, with hash table acceleration.

## Core Responsibilities
- Load external audio files via decoder abstraction, extracting format metadata (bit depth, sample rate, stereo/mono, endianness)
- Maintain a singleton `SoundReplacements` registry with O(1) lookup via hash table
- Provide fallback linear search for hash misses with automatic hash table updates
- Add/update sound option entries by index and slot pair

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ExternalSoundHeader` | class | Extends `SoundHeader` to load and store external audio file metadata and decoded samples |
| `SoundOptions` | struct | File path + decoded audio header; includes static hash function for index/slot pairs |
| `SoundOptionsEntry` | struct | Wrapper pairing a `SoundOptions` with index/slot identifiers for list storage |
| `SoundReplacements` | class | Singleton managing a vector of entries and a hash table for O(1) lookup |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SoundReplacements::m_instance` | `SoundReplacements*` | static | Singleton instance pointer; initialized to 0 |

## Key Functions / Methods

### ExternalSoundHeader::LoadExternal
- **Signature:** `bool LoadExternal(FileSpecifier& File)`
- **Purpose:** Decode an external audio file and populate the header with format metadata.
- **Inputs:** `File` ΓÇô file specifier for the audio to load
- **Outputs/Return:** `bool` ΓÇô true on success, false if decoder unavailable, empty file, or decode fails
- **Side effects:** Allocates buffer via `Load(length)` (inherited); clears buffer on decode failure
- **Calls:** `Decoder::Get()`, decoder methods (`Frames()`, `BytesPerFrame()`, `Decode()`, `IsSixteenBit()`, `IsStereo()`, `IsSigned()`, `IsLittleEndian()`, `Rate()`)
- **Notes:** Fixed-point rate conversion (`FIXED_ONE * decoder->Rate()`); loop boundaries defaulted to 0

### SoundReplacements::SoundReplacements
- **Signature:** `SoundReplacements()`
- **Purpose:** Initialize the singleton instance with empty lists and a zero-filled hash table.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resizes hash table vector to `SoundOptions::HashSize` (256), fills with `NONE` sentinel
- **Calls:** None visible in this file
- **Notes:** Private constructor enforces singleton pattern via `instance()` factory method

### SoundReplacements::GetSoundOptions
- **Signature:** `SoundOptions* GetSoundOptions(short Index, short Slot)`
- **Purpose:** Retrieve sound options by index/slot with hash table acceleration.
- **Inputs:** `Index`, `Slot` ΓÇô sound identifier components
- **Outputs/Return:** Pointer to `SoundOptions::OptionsData` on match; null if not found
- **Side effects:** Updates hash table entry on miss (caches position for future lookups)
- **Calls:** `SoundOptions::HashFunc()`, vector operations
- **Notes:** Hash hit path is O(1); miss triggers linear search O(n) but updates hash for future calls. Reference-based hash access allows in-place update.

### SoundReplacements::Add
- **Signature:** `void Add(const SoundOptions& Data, short Index, short Slot)`
- **Purpose:** Insert or update a sound option entry by index/slot pair.
- **Inputs:** `Data` ΓÇô sound option to store; `Index`, `Slot` ΓÇô identifiers
- **Outputs/Return:** None
- **Side effects:** Modifies or appends to `SOList`; does not update hash table (lazy evaluation in `GetSoundOptions`)
- **Calls:** Vector iteration and operations
- **Notes:** No hash invalidation; relies on linear search + update mechanism in `GetSoundOptions` to rebuild cache on next lookup

## Control Flow Notes
Not inferable from this file alone. This appears to be initialization/loading code (called once or infrequently) rather than frame-loop code. Sound playback and hash lookups likely occur in a separate sound playback module.

## External Dependencies
- `Decoder.h` ΓÇô abstract decoder interface for audio decompression
- `SoundFile.h` (via include) ΓÇô `SoundHeader` base class with buffer allocation (`Load()`) and metadata fields
- `FileHandler.h` (via Decoder.h) ΓÇô `FileSpecifier` type for file I/O
- `cseries.h` (via Decoder.h) ΓÇô standard integer types and platform macros
