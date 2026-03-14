# tools/dumprsrcmap.cpp
## File Purpose
Standalone command-line utility that parses and prints a formatted dump of Macintosh resource map files. Supports AppleSingle, MacBinary, and raw resource fork formats. Used for development/debugging to inspect resource file contents.

## Core Responsibilities
- Accept file path from command-line arguments
- Open and parse resource files via SDL abstraction
- Enumerate all resource types in the file
- For each type, iterate and list all resource IDs with their data sizes
- Format and print resource map to stdout
- Handle resource lifecycle (load/unload) during enumeration

## External Dependencies
- **resource_manager.cpp** (`#include "resource_manager.cpp"`): Provides `open_res_file()`, `get_resource_id_list()`, `get_resource()`, global `cur_res_file_t`, `res_file_t` struct with nested maps
- **FileHandler_SDL.cpp** (`#include "FileHandler_SDL.cpp"`): Provides `FileSpecifier` class, `LoadedResource` class
- **csalerts_sdl.cpp**, **Logging.cpp**, **XML_ElementParser.cpp**: Included for linking but not directly used in this file
- **SDL**: `SDL_RWops` (file I/O abstraction)
- **Standard C**: `<stdio.h>`, `<stdlib.h>`, `<stdarg.h>`
- **Standard C++**: `<vector>`

**Note:** This tool compiles multiple .cpp files as a unit (unusual but valid for standalone utilities). All dynamic allocation and resource enumeration logic is defined in the included files.

# tools/dumpwad.cpp
## File Purpose
Command-line utility to parse and display the contents of Marathon wad files. Reads the wad header, directory metadata, and tag information, then prints a human-readable summary of the level data contained within.

## Core Responsibilities
- Open and read Marathon wad files for inspection
- Parse and validate wad file headers
- Extract and display directory entries with level metadata
- Decode mission flags, environment flags, and entry point information
- Enumerate and list all resource tags within each wad
- Format output for human inspection

## External Dependencies
- **wad.cpp** (included): WAD file reading functions (`open_wad_file_for_reading`, `read_wad_header`, `read_directory_data`, `read_indexed_wad_from_file`, `calculate_directory_offset`)
- **FileHandler_SDL.cpp** (included): File I/O abstraction (`OpenedFile` class, SDL file operations)
- **resource_manager.cpp** (included): Resource file management (included but unused in this tool)
- **crc.cpp** (included): Checksum functions (included for linking)
- **map.h** (header): Map data type definitions (`directory_data`, `directory_entry`, mission/environment/entry-point flag enums)
- **Packing.cpp** (included): Byte serialization macros (`StreamToValue`, `StreamToBytes`)
- **Logging.cpp** (included): Logging support (included for linking)
- **csalerts_sdl.cpp** (included): Error/alert support (dummy stubs for headless operation)

# tools/MapChunkerMain.cpp
## File Purpose
A standalone Mac utility tool that manipulates resource/chunk conversions in Marathon map files (WAD format). Provides a menu-driven interface for reporting on map contents, moving resources into chunks, and extracting chunks back to resources.

## Core Responsibilities
- Macintosh event loop and menu-bar management (Carbon/Classic Mac APIs)
- Reading and writing WAD (wad_data) files with header/directory structures
- Converting between macOS resource fork format and WAD chunk format for PICT (images), CLUT (color tables), sound, and text resources
- Picture and color-table conversion using QuickTime APIs
- File I/O abstraction through FileSpecifier and OpenedFile wrappers
- Reporting chunk/resource inventory to text files

## External Dependencies
- **Includes:** `<Carbon.h>`, `<Movies.h>` (QuickTime); `cseries.h`, `crc.h`, `map.h`, `editor.h`, `FileHandler.h`, `wad.h`
- **Defined elsewhere:** `wad_data`, `wad_header`, `tag_data`, `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `append_data_to_wad()`, `write_wad()`, `write_wad_header()`, `calculate_wad_length()`, Macintosh ToolBox functions (`Gestalt()`, `EnterMovies()`, `MenuSelect()`, etc.)
- **Macintosh APIs:** Resource Manager, File Manager (HGetVol, HSetVol, FSpCreateResFile), QuickTime (QTNewGWorldFromPtr, OpenCPicture), Navigation Services (optional)


