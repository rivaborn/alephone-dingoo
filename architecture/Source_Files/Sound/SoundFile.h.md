# Source_Files/Sound/SoundFile.h

## File Purpose
Defines core sound file management classes for the Aleph One game engine. Provides abstractions for loading, parsing, and accessing sound resources from System 7 sound format files, including support for sound permutations, pitch variation, and runtime custom sound loading.

## Core Responsibilities
- Parse and deserialize System 7 sound format headers (standard and extended variants)
- Manage individual sound sample data with metadata (bit depth, stereo, endianness, loop points, playback rate)
- Organize sounds into definitions with multiple permutations and probabilistic playback selection
- Maintain a hierarchical sound file structure indexed by source and sound index
- Support runtime custom sound addition without modifying the base sound file

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SoundHeader` | class | Encapsulates a single audio sample: format metadata (bitrate, stereo, signed/unsigned, endianness), decoded sample data, and loop point markers |
| `SoundDefinition` | class | Represents a named sound with pitch range variation, playback probability, and up to 5 permutations |
| `SoundFile` | class | Root manager for the entire sound file; maintains 2D vector of sound definitions indexed by source and sound ID |

## Global / File-Static State
None.

## Key Functions / Methods

### SoundHeader::Load (from file)
- Signature: `bool Load(OpenedFile &SoundFile)`
- Purpose: Deserialize and store a System 7 sound resource from an open file
- Inputs: Reference to positioned open file
- Outputs/Return: `true` on success; populates format fields (`sixteen_bit`, `stereo`, `signed_8bit`, `bytes_per_frame`, `little_endian`), loop markers, rate, and allocates `stored_data` buffer
- Side effects: Allocates heap memory via `boost::shared_array`; reads from file
- Calls: `UnpackStandardSystem7Header()`, `UnpackExtendedSystem7Header()`
- Notes: Parses magic numbers to detect header type; stores audio sample data internally

### SoundHeader::Load (from buffer)
- Signature: `bool Load(const uint8* data)`
- Purpose: Parse a System 7 sound from a pre-loaded memory buffer without copying data
- Inputs: Byte pointer to System 7 format sound data
- Outputs/Return: `true` on success; populates metadata but holds reference to input pointer instead of allocating
- Side effects: None (zero-copy; no heap allocation or I/O)
- Calls: `UnpackStandardSystem7Header()`, `UnpackExtendedSystem7Header()`

### SoundHeader::Load (allocate buffer)
- Signature: `uint8* Load(int32 length)`
- Purpose: Pre-allocate a buffer for decoded audio samples
- Inputs: Desired sample count in bytes
- Outputs/Return: Pointer to newly allocated byte array for caller to populate
- Side effects: Clears existing data; allocates via `shared_array<uint8>`; sets `length` field

### SoundFile::Open
- Signature: `bool Open(FileSpecifier &SoundFile)`
- Purpose: Load the sound file header and index all sound definitions
- Inputs: FileSpecifier to the sound resource file
- Outputs/Return: `true` if file is valid; populates `version`, `tag`, `source_count`, `sound_count`, `real_sound_count`, and `sound_definitions` 2D vector
- Side effects: Opens and stores file handle in `opened_sound_file`; allocates sound definition vectors
- Notes: Header size is 260 bytes; does not load individual sound samples yet (lazy loading)

### SoundFile::GetSoundDefinition
- Signature: `SoundDefinition* GetSoundDefinition(int source, int sound_index)`
- Purpose: Retrieve a sound definition by source group and sound ID
- Inputs: Source index, sound index
- Outputs/Return: Pointer to `SoundDefinition` or `nullptr` if out of bounds
- Side effects: None (accessor only)

### SoundFile::Load
- Signature: `void Load(int source, int sound_index)`
- Purpose: Load all permutations for a specific sound definition from disk
- Inputs: Source and sound indices identifying the target definition
- Outputs/Return: None
- Side effects: Reads from file using offsets in the definition; populates `SoundDefinition::sounds` vector with `SoundHeader` objects
- Notes: Called on-demand during runtime; uses `group_offset` and `sound_offsets` from definition

### SoundFile::NewCustomSoundDefinition
- Signature: `int NewCustomSoundDefinition()`
- Purpose: Allocate a new custom sound slot
- Inputs: None
- Outputs/Return: Unique sound index for the new slot
- Side effects: Appends empty `SoundDefinition` to the vector
- Notes: Enables runtime sound addition; returned index must be populated via `AddCustomSoundSlot()`

### SoundFile::AddCustomSoundSlot
- Signature: `bool AddCustomSoundSlot(int index, const char* file)`
- Purpose: Load a sound from an external file into a custom sound slot
- Inputs: Custom sound index and file path
- Outputs/Return: `true` if successfully loaded, `false` on I/O failure
- Side effects: Loads file data and populates the definition at the given index

### SoundFile::UnloadCustomSounds
- Signature: `void UnloadCustomSounds()`
- Purpose: Clear all custom sound data
- Inputs: None
- Outputs/Return: None
- Side effects: Clears custom sound vectors

## Control Flow Notes
Fits into the **initialization and loading phases**:
- `Open()` called at engine startup to index the main sound file
- `Load()` called on-demand during gameplay to populate sound samples
- `GetSoundDefinition()` called at sound-playback time to retrieve definition metadata
- Custom sound methods enable runtime mod/extension without modifying the base file

## External Dependencies
- `AStream.h`: Endian-aware deserialization (uses `AIStreamBE` for big-endian System 7 parsing)
- `FileHandler.h`: Cross-platform file I/O (`OpenedFile`, `FileSpecifier`)
- `<memory>`, `<vector>`: C++ standard library
- `<boost/shared_array.hpp>`: Boost smart array container for sample data
- `cstypes.h`: Custom integer types (`uint8`, `int16`, `int32`, `uint16`, `uint32`, `_fixed`) ΓÇö defined elsewhere
