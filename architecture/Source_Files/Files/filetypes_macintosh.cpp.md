# Source_Files/Files/filetypes_macintosh.cpp

## File Purpose
Manages macOS file type codes (OSType) for the Aleph One game engine, mapping between the engine's abstract `Typecode` enum and macOS 4-byte file type codes. Loads custom typecodes from the resource fork and maintains backward compatibility with Marathon 2 file types through a multi-mapping system.

## Core Responsibilities
- Maintains a static array of OSType codes indexed by Typecode enum values
- Loads custom typecodes from macOS resource fork ('FTyp' resource 128) during initialization
- Builds and maintains a fast lookup map (std::map) from OSType ΓåÆ Typecode for file I/O operations
- Provides bidirectional accessors: Typecode ΓåÆ OSType and OSType ΓåÆ Typecode
- Supports one-to-many mappings (multiple OSTypes can map to one Typecode for M2 compatibility)
- Handles file types for scenarios, savefiles, physics, shapes, sounds, patches, images, preferences, and music

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Typecode | enum (from tags.h) | Abstract engine file type identifiers |
| OSType | macOS typedef | 4-byte file type code |
| file_type_to_a1_typecode_rec | struct | Maps one OSType to one Typecode |
| file_type_to_a1_typecode_t | typedef | std::map<OSType, Typecode> for O(log n) reverse lookup |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| typecodes | OSType[NUMBER_OF_TYPECODES] | static | Primary array: typecodes[Typecode] ΓåÆ OSType |
| additional_typecodes | file_type_to_a1_typecode_rec[] | static | Supplemental mappings (M2 legacy types + A1-specific variants) |
| file_type_to_a1_typecode | std::map<OSType, Typecode> | static | Reverse lookup table; populated at init |

## Key Functions / Methods

### initialize_typecodes
- **Signature:** `void initialize_typecodes()`
- **Purpose:** Initializes typecode mappings; loads custom typecodes from macOS resource fork (if available), then populates the reverse-lookup map.
- **Inputs:** None (reads macOS resource fork via macOS APIs if not SDL_RFORK_HACK)
- **Outputs/Return:** None
- **Side effects:** 
  - Loads 'FTyp' resource 128 (if present and size Γëñ typecodes array size)
  - Clears and repopulates `file_type_to_a1_typecode` map
  - Memory-locks, copies, and unlocks resource data
- **Calls:** GetResource, GetHandleSize, HLock, HUnlock, ReleaseResource (macOS APIs); map::clear, map::operator[]
- **Notes:** Skipped on Mac OS X SDL builds; validates resource size before copying.

### get_typecode
- **Signature:** `OSType get_typecode(Typecode which)`
- **Purpose:** Forward lookup: returns OSType for a given Typecode.
- **Inputs:** `which` (Typecode enum index)
- **Outputs/Return:** OSType, or '????' for out-of-range indices
- **Side effects:** None
- **Calls:** None
- **Notes:** Bounds-checked; returns sentinel '????' if which < 0 or which ΓëÑ NUMBER_OF_TYPECODES.

### set_typecode
- **Signature:** `void set_typecode(Typecode which, OSType _type)`
- **Purpose:** Sets OSType for a given Typecode (allows runtime override of typecodes).
- **Inputs:** `which` (Typecode index), `_type` (new OSType)
- **Outputs/Return:** None
- **Side effects:** Modifies global typecodes[] array
- **Calls:** None
- **Notes:** Bounds-checked; silently does nothing if out of range.

### get_typecode_for_file_type
- **Signature:** `Typecode get_typecode_for_file_type(OSType inType)`
- **Purpose:** Reverse lookup: returns Typecode for a given OSType (used during file I/O to identify file purpose).
- **Inputs:** `inType` (macOS OSType code)
- **Outputs/Return:** Typecode, or `_typecode_unknown` if not in map
- **Side effects:** None
- **Calls:** map::find, assert
- **Notes:** Uses pre-built map for O(log n) lookup; asserts map is non-empty (requires initialize_typecodes to be called first).

### get_all_file_types_for_typecode
- **Signature:** `const vector<OSType> get_all_file_types_for_typecode(Typecode which)`
- **Purpose:** Returns all OSTypes that map to a given Typecode (handles one-to-many mappings).
- **Inputs:** `which` (Typecode enum value)
- **Outputs/Return:** std::vector<OSType> (may be empty)
- **Side effects:** Creates new vector (no global mutation)
- **Calls:** vector::push_back
- **Notes:** Searches primary typecodes array first (if valid and not '????'), then additional_typecodes array for all matching entries.

## Control Flow Notes
- **Init phase:** `initialize_typecodes()` must be called early in engine startup before any file I/O operations.
- **Runtime:** File type lookups via `get_typecode_for_file_type()` occur during file dialogs, resource loading, and preference/environment file discovery.
- **Shutdown:** No cleanup needed (all data is static).
- Code is conditionally compiled on `mac` or `(SDL && SDL_RFORK_HACK)`; file type mapping is macOS-specific.

## External Dependencies
- **Carbon.h** (macOS APIs): Resource fork management (GetResource, HLock, HUnlock, ReleaseResource, GetHandleSize)
- **CSeries/csalerts.h:** Assert macro (used in get_typecode_for_file_type)
- **tags.h:** Typecode enum definition and function declarations
- **std::map, std::vector:** STL containers
