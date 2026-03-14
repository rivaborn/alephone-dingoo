# Source_Files/Sound/ReplacementSounds.h

## File Purpose
Manages external sound file replacements specified via MML configuration. Provides a singleton registry (`SoundReplacements`) to load and retrieve custom sound files keyed by sound index and permutation slot, with a hash table for efficient lookup.

## Core Responsibilities
- Define `ExternalSoundHeader` class for loading sound data from external files (extends `SoundHeader`)
- Store sound replacement metadata (file path, parsed header) in `SoundOptions` struct
- Implement singleton `SoundReplacements` registry with hash-based lookup by sound index and slot
- Provide hash function to avoid collisions between sounds with same index but different slots
- Support runtime addition and reset of replacement sounds

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ExternalSoundHeader` | class | Extends `SoundHeader` to add `LoadExternal()` for file-based sound loading |
| `SoundOptions` | struct | Holds file path, parsed sound header, and static hash function for index/slot lookups |
| `SoundOptionsEntry` | struct | Maps a (Index, Slot) pair to its `SoundOptions` data |
| `SoundReplacements` | class | Singleton registry managing a vector-based collection and hash table of sound replacements |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SoundReplacements::m_instance` | `SoundReplacements*` | static | Singleton instance pointer |
| `SoundReplacements::SOList` | `std::vector<SoundOptionsEntry>` | member | Linear storage of all registered sound replacements |
| `SoundReplacements::SOHash` | `std::vector<int16>` | member | Hash table for O(1) lookup of sound options by hashed index/slot |

## Key Functions / Methods

### ExternalSoundHeader::LoadExternal
- **Signature:** `bool LoadExternal(FileSpecifier& File)`
- **Purpose:** Load sound data from an external file (likely WAV, AIFF, or other format)
- **Inputs:** File reference to external sound file
- **Outputs/Return:** `bool` ΓÇö success/failure
- **Side effects:** Populates inherited `SoundHeader` fields (sample rate, channels, bit depth, loop points, audio data buffer)
- **Calls:** (Implementation in .cpp file; likely uses `SoundFile` or file I/O utilities)
- **Notes:** Virtual destructor in parent `SoundHeader`; no-op constructor delegates to parent

### SoundReplacements::instance
- **Signature:** `static inline SoundReplacements* instance()`
- **Purpose:** Lazy-initialize and return singleton instance
- **Inputs:** None
- **Outputs/Return:** Pointer to global `SoundReplacements` instance
- **Side effects:** Creates instance on first call; subsequent calls return cached pointer
- **Calls:** `new SoundReplacements` (destructor private; never deleted)
- **Notes:** Inline implementation; classic Meyer's singleton pattern variant

### SoundReplacements::GetSoundOptions
- **Signature:** `SoundOptions* GetSoundOptions(short Index, short Slot)`
- **Purpose:** Retrieve sound replacement options by sound index and permutation slot
- **Inputs:** Index (sound ID), Slot (permutation variant)
- **Outputs/Return:** Pointer to `SoundOptions` or null if not found
- **Side effects:** None (read-only lookup via hash table)
- **Calls:** Hash function internally
- **Notes:** Implementation in .cpp; likely uses `SOHash` for O(1) collision-free lookup

### SoundReplacements::Add
- **Signature:** `void Add(const SoundOptions& Data, short Index, short Slot)`
- **Purpose:** Register a new replacement sound under given index and slot
- **Inputs:** Sound options (file + header), Index, Slot
- **Outputs/Return:** None
- **Side effects:** Appends entry to `SOList`, updates `SOHash` table
- **Calls:** `SoundOptions::HashFunc()` to compute hash key
- **Notes:** No duplicate checking visible; implementation detail in .cpp

### SoundOptions::HashFunc
- **Signature:** `static inline uint8 HashFunc(short Index, short Slot)`
- **Purpose:** Compute collision-free hash key for (Index, Slot) pair
- **Inputs:** Index and Slot (both 16-bit signed integers)
- **Outputs/Return:** 8-bit hash value
- **Side effects:** None (pure function)
- **Calls:** Bitwise XOR and left-shift
- **Notes:** XOR of Index and (Slot << 4) ensures sounds with same Index but different Slots map to different buckets; `HashMask = 255` limits result to 256 buckets

## Control Flow Notes
Initialization-phase component. `SoundReplacements` is populated during game startup (likely from MML script parsing) before sounds are played. `GetSoundOptions()` is called at runtime to look up custom sounds; `Reset()` clears all entries (likely on map/scenario unload).

## External Dependencies
- **Includes:** `<string>`, `"SoundFile.h"` (bundled)
- **Base classes:** `SoundHeader` (from `SoundFile.h`)
- **Types used:** `std::string`, `std::vector`, `FileSpecifier` (from `FileHandler.h` via `SoundFile.h`)
- **Defined elsewhere:** `FileSpecifier`, `SoundHeader`, `uint8`, `int16`, `int32`
