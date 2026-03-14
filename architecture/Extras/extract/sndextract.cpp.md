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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `sound_file_header` | struct | Binary file header; contains version, tag, source/sound counts |
| `sound_definition` | struct | Metadata for one sound: offsets, lengths, permutation count, sound code |
| `SoundHeader` / `ExtSoundHeader` | struct | Mac OS sound format headers (standard and extended) |
| `Handle` | typedef | Mac OS opaque resource handle |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sound_definitions` | `struct sound_definition[]` | static (STATIC_DEFINITIONS macro) | Array of sound definitions, copied and modified during extraction |

## Key Functions / Methods

### main
- Signature: `void main(int argc, char **argv)`
- Purpose: Entry point; validates command-line arguments and invokes build process.
- Inputs: `argc`, `argv` (command-line arguments)
- Outputs/Return: Exits with status 0 on success, 1 on error.
- Side effects: Calls `build_sounds_file`; writes to stderr on invalid arguments.
- Calls: `build_sounds_file`, `fprintf`, `exit`
- Notes: Requires at least 3 arguments (program name + source + destination).

### build_sounds_file
- Signature: `void build_sounds_file(char *destination_filename, short source_count, char **source_filenames)`
- Purpose: Creates output sound file; manages resource file opening, definition extraction, and file layout.
- Inputs: Destination filename, count and array of source filenames.
- Outputs/Return: Writes binary sound file to disk.
- Side effects: Allocates 2 ├ù `NUMBER_OF_SOUND_DEFINITIONS` blocks; opens/closes Mac resource files; writes to destination file.
- Calls: `fopen`, `malloc`, `memset`, `fwrite`, `strcpy`, `c2pstr`, `OpenResFile`, `extract_sound_resources`, `CloseResFile`, `free`, `fclose`, `fprintf`, `exit`
- Notes: First source file is treated as primary; subsequent sources use it as fallback for missing sounds (via `original_definitions` parameter).

### extract_sound_resources
- Signature: `static void extract_sound_resources(FILE *stream, long *definition_offset, short base_resource, struct sound_definition *original_definitions, struct sound_definition *working_definitions)`
- Purpose: Extracts sound resources from a Mac resource file for each sound definition; writes audio data and updates metadata.
- Inputs: Output stream, definition offset pointer, base resource ID, original definitions (nullable, for fallback), working definitions to populate.
- Outputs/Return: Modifies `stream`, `definition_offset`, and `working_definitions`; no return value.
- Side effects: Seeks in output stream; writes sound data and definition arrays; allocates/locks/releases Mac handles.
- Calls: `fseek`, `ftell`, `GetResource`, `HLock`, `GetSoundHeaderOffset`, `fwrite`, `ReleaseResource`
- Notes: Iterates permutations per sound; uses `ALIGN_LONG` macro to pad each sound to 4-byte boundary; if no permutations found and `original_definitions` is non-null, copies sound from primary source.

## Control Flow Notes
**Initialization ΓåÆ Execution ΓåÆ Shutdown:**
1. **main**: Parses CLI arguments; entry point.
2. **build_sounds_file**: Allocates buffers, writes file header + initial definition arrays, iterates source files.
3. For each source:
   - Opens Mac resource file.
   - Calls **extract_sound_resources** with base_resource=0 (primary) or 10000 (alternate).
   - Primary: initializes `original_definitions` for fallback; alternates use it.
4. **extract_sound_resources**: Writes audio data to end of file, updates definition metadata, seeks back to write definitions.
5. Cleanup: Free buffers, close files, exit.

## External Dependencies
- **Macintosh headers** (`macintosh_cseries.h`): `OpenResFile`, `CloseResFile`, `GetResource`, `HLock`, `ReleaseResource`, `ResError`, `GetSoundHeaderOffset`, `c2pstr`, `Str255`, `SndListHandle`
- **Game engine headers**: `::world.h`, `::mysound.h`, `::sound_definitions.h` (defines `sound_definition`, `NUMBER_OF_SOUND_DEFINITIONS`, `NUMBER_OF_SOUND_SOURCES`, constants like `SOUND_FILE_VERSION`, `SOUND_FILE_TAG`)
- **Standard C**: `<stdio.h>` (fopen, fprintf, fwrite, fseek, ftell, fclose), `<stdlib.h>` (malloc, free, exit), `<string.h>` (strcpy, memset, memcpy)
- **Local headers**: `byte_swapping.h` (included but not visibly used in this file)
