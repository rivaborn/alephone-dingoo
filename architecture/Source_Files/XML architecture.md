# Subsystem Overview

## Purpose

The XML subsystem provides configuration file parsing infrastructure for the Aleph One game engine using the Expat C XML parser library. It implements a hierarchical element parser tree with type-safe value reading utilities and platform-specific data adapters (file I/O, memory buffers, macOS resource forks). Game-specific parsers integrate into this tree to load damage definitions, colors, shapes, and level-specific scripts from XML-encoded configuration data.

## Key Files

| File | Role |
|------|------|
| XML_ElementParser.h/cpp | Base class for hierarchical element parsing; provides type-safe value readers (int16/32, uint16/32, float, boolean) with bounds checking, UTF-8 conversion, and error reporting utilities |
| XML_Configure.h/cpp | Abstract base class orchestrating Expat parser lifecycle, element tree traversal, static callback dispatch, and error collection across read/parse/interpretation phases |
| XML_DataBlock.h/cpp | Adapter for parsing XML from in-memory buffers; delegates parsing to parent class with buffer-based data retrieval |
| XML_Loader_SDL.h/cpp | Concrete adapter for parsing XML from disk files via SDL FileHandler abstraction; supports single-file and directory-wide parsing with error reporting |
| XML_ResourceFork.h/cpp | macOS-specific adapter for parsing XML resources from resource forks via Resource Manager API |
| XML_ParseTreeRoot.h | Declares global `RootParser` object and provides `SetupParseTree()` initialization and `ResetAllMMLValues()` reset entry points |
| XML_MakeRoot.cpp | Initializes complete parser tree by attaching subsystem-specific parsers as children of root "marathon" element |
| XML_LevelScript.h/cpp | Manages level-specific XML scripts (MML, Lua, music, movies, load screens) loaded from map file resource 128; executes scripts at lifecycle points (level start, end, restoration) |
| ColorParser.h/cpp | Game-specific parser for `<color>` XML elements; converts RGB float attributes [0ΓÇô1] to 16-bit values and stores in caller-provided array |
| DamageParser.h/cpp | Game-specific parser for `<damage>` XML elements; populates `damage_definition` structures with type, flags, and scaling parameters |
| ShapesParser.h/cpp | Game-specific parser for `<shape>` XML elements; composes collection, CLUT, sequence, and frame indices into shape descriptor values |

## Core Responsibilities

- Parse XML configuration files from multiple data sources (files, memory buffers, resource forks) via pluggable adapter pattern
- Implement hierarchical element parser tree with parent-child relationships and virtual lifecycle hooks (Start, End, HandleAttribute, HandleString, ResetValues)
- Provide type-safe value parsing utilities with bounds validation: integers (int16/32, uint16/32), floats, booleans, and UTF-8-to-MacRoman string conversion
- Delegate element-specific parsing to game subsystem parsers (colors, damage, shapes) integrated as children in the element tree
- Load and execute level-specific scripts (MML, Lua, music, movies) from map file resources, with pseudo-level support (Default, Restore, End)
- Collect and report three error categories (read errors, XML parse errors, interpretation errors) with line numbers and source context; throttle error output to prevent log spam
- Manage case-insensitive XML tag/attribute matching and validate attribute values against engine limits (collection counts, CLUT bounds, damage types)

## Key Interfaces & Data Flow

**Exposes to game engine:**
- `XML_ElementParser` base class for inheritance by subsystem-specific parsers
- Factory functions for element parsers: `Color_GetParser()`, `Shape_GetParser()`, `DamageParser()` (called by parent tree during configuration load)
- `RootParser` and `SetupParseTree()` for XML system initialization during engine startup
- `ResetAllMMLValues()` for resetting all MML configuration to hardcoded defaults
- `XML_LevelScript::Load()` interface for per-level script loading from map resources

**Consumes from other subsystems:**
- Expat XML parser library (C API): `XML_Parser`, `XML_Parse()`, `XML_SetUserData()`, callback registration
- FileHandler (cseries.h) for file I/O abstraction: `FileSpecifier`, `OpenedFile`, directory enumeration
- Images subsystem: `get_text_resource_from_scenario()` for loading embedded resources
- Lua subsystem: `LoadLuaScript()` for Lua code execution
- Music subsystem: `Music::instance()` for level music management
- OGL_LoadScreen subsystem: `OGL_LoadScreen::instance()` for load screen display
- cseries.h platform abstractions: macros (`PIN()`), string utilities (`StringsEqual()`), logging (`logAnomaly()`)

## Runtime Role

`SetupParseTree()` is called at engine startup to construct the global parser tree, attaching all subsystem parsers as children of the root "marathon" element. When configuration files are loaded, `XML_Loader_SDL` (or platform-specific adapters) instantiate and call `DoParse()` on the root parser:

1. Expat XML parser is created with static callback handlers
2. Data is read in chunks via virtual `GetData()` method
3. Expat fires parser events (element start/end, attributes, character data)
4. Static callbacks dispatch events to instance methods and element tree
5. Element handlers call user-defined Start/End/HandleAttribute/HandleString hooks
6. Parse errors and interpretation errors are collected and reported

Level scripts execute per-level: `XML_LevelScript::Load()` reads resource 128 from the map file, parses XML commands, and executes MML/Lua scripts at appropriate lifecycle points (level start, level end, game restoration). The subsystem remains active throughout gameplay to support dynamic configuration updates.

## Notable Implementation Details

- Expat integration pattern: C library callbacks are wrapped as static methods forwarding to instance methods via `XML_SetUserData()`
- Element tree traversal maintains a stack (`CurrentElement` pointer) of the active element during nested XML parsing
- Bounded integer validation uses `PIN(value, min, max)` macro for range clamping and error messages via `ReadBoundedInt16Value()` and similar utilities
- UTF-8 to MacRoman conversion implements a full state machine in `DeUTF8*()` functions for cross-platform compatibility
- Error throttling limits output to 7 errors per parse session to prevent log spam on malformed files
- Singleton parser instances (e.g., `ColorParser`) allow pointer reassignment for multi-element parsing
- Platform-specific paths: macOS uses `XML_ResourceFork` (Carbon/classic Mac API), SDL uses `XML_Loader_SDL` (file I/O)
- Pseudo-level support via `XML_LevelScript`: Default (global config), Restore (reset on chapter/failure), End (end-of-game scripts)
