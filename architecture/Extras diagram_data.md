# Extras/extract/shapeextract.cpp
## File Purpose
Command-line utility for extracting shape resources from Macintosh resource files. Reads `.256` resource types from a Mac resource fork and writes structured shape collection data to a binary output file for engine consumption.

## Core Responsibilities
- Parse and validate command-line arguments (source/destination paths)
- Open and manage Macintosh resource files
- Iterate through shape collection IDs and extract resource data
- Write collection headers and resource offsets/lengths to output binary
- Handle resource memory and file I/O errors

## External Dependencies
- **Macintosh Toolbox:** `OpenResFile`, `CloseResFile`, `GetResource`, `HLock`, `ReleaseResource`, `ResError`, `GetHandleSize`
- **Standard C:** `FILE`, `fopen`, `fclose`, `fwrite`, `fseek`, `strcpy`, `fprintf`, `exit`
- **Custom headers:** `macintosh_cseries.h` (platform macros/defs), `shape_descriptors.h`, `shape_definitions.h`
- **Mac string utilities:** `c2pstr` (CΓåÆPascal string conversion)

# Extras/extract/sndextract.cpp
## File Purpose
Command-line utility for extracting sound resources from Macintosh resource files and combining them into a single binary sound archive. Reads sound definitions and audio data ('snd ' resources) from one or more Mac resource files and writes them to a structured output file with header, definition tables, and raw audio data.

## Core Responsibilities
- Parse and validate command-line arguments (source and destination filenames)
- Open and manage Macintosh resource files
- Extract sound resources and their permutations (variations) from Mac resources
- Compute sound data offsets and sizes with proper alignment
- Write output binary file with header, definition arrays, and concatenated audio data
- Support multi-source merging with fallback: alternate sources inherit missing sounds from primary source

## External Dependencies
- **Macintosh headers** (`macintosh_cseries.h`): `OpenResFile`, `CloseResFile`, `GetResource`, `HLock`, `ReleaseResource`, `ResError`, `GetSoundHeaderOffset`, `c2pstr`, `Str255`, `SndListHandle`
- **Game engine headers**: `::world.h`, `::mysound.h`, `::sound_definitions.h` (defines `sound_definition`, `NUMBER_OF_SOUND_DEFINITIONS`, `NUMBER_OF_SOUND_SOURCES`, constants like `SOUND_FILE_VERSION`, `SOUND_FILE_TAG`)
- **Standard C**: `<stdio.h>` (fopen, fprintf, fwrite, fseek, ftell, fclose), `<stdlib.h>` (malloc, free, exit), `<string.h>` (strcpy, memset, memcpy)
- **Local headers**: `byte_swapping.h` (included but not visibly used in this file)

# Extras/noresnames.cpp
## File Purpose
Command-line utility that strips resource names from all resources in Mac OS Classic resource files. Takes file paths as arguments and iterates through each file, clearing the name field of every resource.

## Core Responsibilities
- Parse command-line file path arguments and validate filename length
- Open resource forks of Mac OS resource files
- Enumerate all resource types within a file
- Enumerate all individual resources for each type
- Retrieve and clear resource names (set to empty string)
- Handle file-level and resource-level errors gracefully

## External Dependencies
- `<Resources.h>`: Mac OS Classic Resource Manager API (`FSMakeFSSpec`, `FSpOpenResFile`, `Count1Types`, `Get1IndResource`, `SetResInfo`, etc.)
- `<stdio.h>`: Standard I/O (`fprintf`)
- `<string.h>`: String utilities (`strlen`, `memcpy`)

# Extras/physics_patches.cpp
## File Purpose
A standalone command-line utility that compares two physics WAD (game data) files and generates a binary patch file containing only the differences. Used for creating incremental physics updates without redistributing entire physics files.

## Core Responsibilities
- Parse command-line arguments (original WAD, new WAD, output patch filename)
- Load two physics WAD files from disk and validate their versions
- Perform byte-by-byte comparison of definition data across all game types
- Identify contiguous regions of changes (deltas) for each definition type
- Create a new WAD containing only changed data with parent checksum metadata
- Write patch file to disk with versioning information

## External Dependencies
- **macintosh_cseries.h** ΓÇö Mac classic file APIs (`FSMakeFSSpec`, `FSSpec`, `FileDesc`, `c2pstr`, `OSErr`)
- **wad.h** ΓÇö WAD file I/O (`open_wad_file_for_reading`, `read_wad_header`, `read_indexed_wad_from_file`, `extract_type_from_wad`, `append_data_to_wad`, `create_empty_wad`, `fill_default_wad_header`, `write_wad_header`, `write_directorys`, `calculate_and_store_wadfile_checksum`)
- **extensions.h** ΓÇö Game engine definitions; defines `definitions` array with type metadata (tag, etc.); provides `NUMBER_OF_DEFINITIONS`
- **map.h, effects.h, projectiles.h, monsters.h, weapons.h, items.h, media.h** ΓÇö Game data headers (included to allow extensions.h to be included)
- **tags.h** ΓÇö Tag type constants (transitively via wad.h)
- **stdio.h** (implicit) ΓÇö `fprintf`, `stderr`
- **string.h** (implicit) ΓÇö `strcpy`


