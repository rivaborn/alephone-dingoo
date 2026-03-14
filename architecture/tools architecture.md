# Subsystem Overview

## Purpose
Development and debugging utilities for inspecting and manipulating Marathon resource and wad files. They provide command-line and GUI interfaces to examine file contents, convert between formats, and report on map and resource inventory.

## Key Files
| File | Role |
|------|------|
| tools/dumprsrcmap.cpp | Command-line utility to parse and dump Macintosh resource map files (AppleSingle, MacBinary, raw formats) |
| tools/dumpwad.cpp | Command-line utility to parse and display Marathon wad file contents, headers, and directory metadata |
| tools/MapChunkerMain.cpp | Macintosh GUI utility for converting between resource fork and WAD chunk formats; manipulates PICT, CLUT, sound, and text resources |

## Core Responsibilities
- Parse and enumerate resource files in multiple formats (AppleSingle, MacBinary, raw resource forks)
- Inspect Marathon wad file structure, headers, and directory entries
- Decode and display mission flags, environment flags, and entry point information from wad metadata
- Convert between macOS resource fork format and WAD chunk format for images, color tables, sound, and text
- Provide command-line and menu-driven interfaces for file inspection and resource inventory reporting

## Key Interfaces & Data Flow
**Consumes:**
- resource_manager.cpp (resource enumeration, file loading)
- wad.cpp (wad file reading, header/directory parsing)
- FileHandler_SDL.cpp (file I/O abstraction, FileSpecifier and OpenedFile classes)
- map.h (data structures: directory_data, directory_entry, flag enums)
- Packing.cpp (byte serialization macros)

**Exposes:**
- Formatted stdout output for resource and wad inspection
- Modified wad files (MapChunkerMain output)

## Runtime Role
These tools are not runtime components. They execute during development and debugging phases outside the main game loop to inspect file contents, validate structures, and manipulate Marathon data files.

## Notable Implementation Details
- dumprsrcmap and dumpwad compile multiple .cpp files as a single translation unit (unusual but valid for standalone utilities)
- MapChunkerMain is platform-specific to macOS (Carbon APIs and QuickTime) and uses menu-bar event loops
- dumpwad validates wad headers and decodes mission/environment/entry-point flag fields
- FileHandler_SDL abstraction provides portability for command-line tools across platforms
