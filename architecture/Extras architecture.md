# Subsystem Overview

## Purpose
Build-time utilities for extracting and processing Macintosh resource files into binary data archives consumed by the game engine. Includes tools for shape extraction, sound resource compilation, resource name stripping, and physics patch generation.

## Key Files
| File | Role |
|------|------|
| `Extras/extract/shapeextract.cpp` | Command-line utility to extract shape collection resources from Mac `.256` resource fork files and write structured binary output |
| `Extras/extract/sndextract.cpp` | Command-line utility to extract sound definitions and audio ('snd ' resources) from Mac resource files, merge multiple sources, and write combined sound archive |
| `Extras/noresnames.cpp` | Command-line utility to strip all resource names from Mac OS resource files |
| `Extras/physics_patches.cpp` | Command-line utility to compare two physics WAD files, identify deltas, and generate incremental patch files |

## Core Responsibilities
- Parse and validate command-line arguments (input/output file paths)
- Open and enumerate Macintosh resource files using Mac OS Classic Resource Manager APIs
- Extract game resource data (shapes, sounds, physics definitions) from Mac resource forks
- Compute binary layout (offsets, sizes, alignment) for output archives
- Write structured binary output files with headers, definition tables, and raw data payloads
- Support multi-source resource merging with fallback inheritance (sndextract)
- Perform byte-level comparison and delta detection for incremental updates (physics_patches)

## Key Interfaces & Data Flow
- **Consumes:** Macintosh resource files (`.rsrc` forks with `.256`, `snd `, `snd#`, `PHYS` resources); game engine header definitions from `Source_Files/` (world.h, mysound.h, sound_definitions.h, extensions.h, wad.h)
- **Produces:** Binary data archives (shape collections, sound archives, physics patches) consumed at engine runtime and by WAD packaging pipeline
- **Dependency chain:** Mac OS Classic Resource Manager (via `macintosh_cseries.h`) ΓåÆ resource extraction ΓåÆ binary serialization ΓåÆ engine consumption

## Runtime Role
Not a runtime subsystem. These are off-line development/build tools executed during asset preparation, not during game init/frame/shutdown cycles. Output files are static data consumed by the main engine at load time.

## Notable Implementation Details
- **shapeextract**: Uses Macintosh Toolbox APIs (`OpenResFile`, `GetResource`, `HLock`, `ReleaseResource`) and CΓåöPascal string conversion (`c2pstr`) for Mac compatibility.
- **sndextract**: Writes aligned audio data with header/definition metadata; supports inheritance fallback where alternate sources provide missing sound permutations if primary source lacks them.
- **physics_patches**: Generates versioned WAD files with parent checksum metadata; enables distribution of incremental physics updates rather than full WAD redistribution.
- **noresnames**: Direct Resource Manager enumeration (Count1Types, Get1IndResource, SetResInfo) for bulk metadata stripping.
