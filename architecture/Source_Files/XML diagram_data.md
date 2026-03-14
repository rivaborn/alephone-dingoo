# Source_Files/XML/ColorParser.cpp
## File Purpose
An XML element parser for color definitions in the Aleph One game engine. Parses `<color>` XML elements with red, green, and blue float attributes, converts them to 16-bit RGB values, and stores them in a caller-provided color array at a specified or default index.

## Core Responsibilities
- Parse XML `<color>` element attributes (red, green, blue, optional index)
- Validate that all required attributes are present in the element
- Convert floating-point color channel values [0ΓÇô1] to 16-bit unsigned integers
- Clamp color values to valid range [0, 65535]
- Store parsed colors into a caller-supplied `rgb_color` array
- Support both indexed and non-indexed color storage modes
- Provide a singleton parser instance for reuse across multiple color element parses

## External Dependencies
- **Includes**: `<string.h>`, `"ColorParser.h"`, (transitively: `"XML_ElementParser.h"`, `"cseries.h"`)
- **Helper functions** (defined elsewhere): `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `UnrecognizedTag()`, `AttribsMissing()`
- **Types** (defined elsewhere): `XML_ElementParser` (base class), `rgb_color` (16-bit RGB struct), `uint16`
- **Macros** (defined elsewhere): `PIN()` (likely bounds-clamp: `PIN(val, min, max)`)

# Source_Files/XML/ColorParser.h
## File Purpose
Header file declaring an XML parser interface for color elements in game configuration files. Provides factory and configuration functions for parsing color data (RGB values) into arrays, supporting both indexed and non-indexed color storage.

## Core Responsibilities
- Factory function to instantiate a color XML element parser
- Configuration of target color array for parsed values
- Support for optional indexed color array storage (index attribute only when NumColors > 0)

## External Dependencies
- **Include**: `cseries.h` (core types, SDK)
- **Include**: `XML_ElementParser.h` (parser base class)
- **Symbols used**: `rgb_color` type, `XML_ElementParser` class (both defined elsewhere)

# Source_Files/XML/DamageParser.cpp
## File Purpose
Implements an XML parser for damage definition elements in the Aleph One game engine. It parses damage configuration attributes (type, flags, base, random, scale) from XML and populates a damage_definition structure. Part of a larger XML configuration system.

## Core Responsibilities
- Parse and validate individual XML damage element attributes
- Convert attribute values to appropriate types (int16, float, fixed-point)
- Enforce value constraints via bounded validation
- Provide a singleton parser instance accessible to the XML parsing system
- Support multiple damage definitions by allowing pointer reassignment

## External Dependencies
- **Framework:** `XML_ElementParser` (base class, defined elsewhere)
- **Utilities:** `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`, `ReadBoundedNumericalValue()`, `UnrecognizedTag()` (defined elsewhere, likely in XML parsing framework)
- **Constants:** `NONE`, `NUMBER_OF_DAMAGE_TYPES`, `FIXED_ONE`, `SHRT_MIN`, `SHRT_MAX` (defined elsewhere)
- **Includes:** `cseries.h` (umbrella header), `DamageParser.h` (public interface), standard C libs (`string.h`, `limits.h`)

# Source_Files/XML/DamageParser.h
## File Purpose
Declares the XML parser interface for damage element definitions. Parses `<damage>` XML elements and populates `damage_definition` structures with damage type, behavior flags, and scaling parameters. Part of the Aleph One engine's data-driven configuration system.

## Core Responsibilities
- Provide factory function for damage element parser creation
- Set the target damage structure to be populated by parsed XML
- Enable XML parsing of damage attributes (type, flags, base, random, scale)

## External Dependencies
- **map.h**: Defines `damage_definition` struct, damage type enum (`_damage_explosion`, etc.), and damage flag enum (`_alien_damage`).
- **XML_ElementParser.h**: Base parser class providing attribute/element handling framework.
- Aleph One game engine internals (implementation in corresponding `.cpp`).

# Source_Files/XML/ShapesParser.cpp
## File Purpose
Parses XML `<shape>` elements to extract and validate shape descriptor attributes (collection, CLUT, sequence/frame indices), then composes them into a `shape_descriptor` value. Provides a reusable static parser that multiple callers can configure via pointer/flag injection.

## Core Responsibilities
- Parse XML shape element attributes: `coll`, `clut`, `seq`, and `frame`
- Validate attribute values against engine limits (collection, CLUT, and sequence bounds)
- Compose validated attributes into a shape descriptor using `BUILD_DESCRIPTOR()` and `BUILD_COLLECTION()`
- Support flexible validation: allow `NONE` (uninitialized) values conditionally via `NONE_Is_OK`
- Provide static parser instance and public interface functions for multi-call reuse
- Report parsing errors via `UnrecognizedTag()` and `AttribsMissing()` callbacks

## External Dependencies
- **Includes**: `cseries.h` (utility macros/functions), `ShapesParser.h` (public interface)
- **Base class**: `XML_ElementParser` (defined elsewhere; provides callback framework)
- **Types**: `shape_descriptor`, `uint16` (defined elsewhere)
- **Functions called (defined elsewhere)**: `StringsEqual()`, `ReadBoundedUInt16Value()`, `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()`, `UnrecognizedTag()`, `AttribsMissing()`
- **Constants (defined elsewhere)**: `MAXIMUM_COLLECTIONS`, `MAXIMUM_CLUTS_PER_COLLECTION`, `MAXIMUM_SHAPES_PER_COLLECTION`, `UNONE`

# Source_Files/XML/ShapesParser.h
## File Purpose
Header file providing an XML parser factory and configuration interface for parsing shape elements from XML configuration files. Enables multiple parser instances to populate shape_descriptor values with optional "NONE" value support.

## Core Responsibilities
- Provide a factory function (`Shape_GetParser()`) to obtain XML element parsers for shape data
- Configure target pointers where parsed shape descriptors should be written
- Enforce optional "NONE" acceptance policy per parser instance
- Support multiple concurrent parser instances for different shape elements

## External Dependencies
- `#include "shape_descriptors.h"` ΓÇô Defines `shape_descriptor` typedef and bitfield macros (`GET_DESCRIPTOR_SHAPE`, `BUILD_DESCRIPTOR`, etc.)
- `#include "XML_ElementParser.h"` ΓÇô Defines `XML_ElementParser` base class with virtual parsing hooks (Start, End, HandleAttribute, HandleString, ResetValues)
- Aleph One game engine framework (Bungie Studios GPL project)

# Source_Files/XML/XML_Configure.cpp
## File Purpose
Implementation of XML configuration file parser for the Marathon/Aleph One game engine. Orchestrates parsing via the Expat XML library, manages element state transitions, and collects interpretation errors during config file validation.

## Core Responsibilities
- Provide static callback wrappers that forward parser events to instance methods
- Manage XML element tree traversal (StartElement finds/activates children, EndElement traverses back to parent)
- Call user-defined element handlers (Start, End, HandleAttribute, HandleString) at appropriate parse points
- Perform parsing loop: repeatedly read data chunks via virtual GetData(), feed to Expat, handle parse errors
- Format and report interpretation errors (validation failures on attributes, structure, etc.)

## External Dependencies
- **Expat XML library:** `#include "expat.h"` ΓÇö provides XML_Parser, XML_ParserCreate(), XML_Parse(), XML_ErrorString(), XML_GetErrorCode(), XML_GetCurrentLineNumber()
- **XML_ElementParser** (defined elsewhere): `#include "XML_ElementParser.h"` ΓÇö element tree node type with handlers (Start, End, HandleAttribute, HandleString, NameMatch, FindChild, GetName, ErrorString member)
- **Standard library:** stdio.h (unused here), stdarg.h (for va_list/vsprintf in ComposeInterpretError)
- **cseries.h** (project utility headers)

# Source_Files/XML/XML_Configure.h
## File Purpose

Abstract base class that orchestrates XML file parsing for Marathon engine configuration. Integrates the Expat C parser library with a tree of XML_ElementParser objects to load and apply game configuration from XML files. Subclasses implement data source (GetData) and error handling.

## Core Responsibilities

- Manage Expat XML parser lifecycle and state (create, feed data, destroy)
- Adapt C-style Expat callbacks to C++ instance methods
- Delegate element-specific parsing to XML_ElementParser element tree
- Track and report three categories of errors: read errors, XML parse errors, and interpretation errors
- Maintain a stack of the current active element parser during document traversal
- Coordinate parsing across multiple data chunks (for streaming/large file support)

## External Dependencies

- **expat.h** ΓÇô Expat XML parser library: provides XML_Parser type, XML_Parse, XML_SetUserData, callback typedefs (XML_StartElementHandler, XML_EndElementHandler, XML_CharacterDataHandler)
- **XML_ElementParser.h** ΓÇô Element parser base class (CurrentElement points to instances of this type; method calls: Start, End, HandleAttribute, HandleString, FindChild)
- **Standard C++** ΓÇô constructor, destructor, virtual methods, includes for stdlib functionality (handled by other headers)

# Source_Files/XML/XML_DataBlock.cpp
## File Purpose
Implements the `XML_DataBlock` class, a memory-based XML parser adapter that handles data block parsing and provides platform-specific error reporting. Manages read errors, XML parse errors, and interpretation errors with different handling for macOS (Classic/Carbon) and SDL platforms.

## Core Responsibilities
- Retrieves and validates XML data from in-memory buffers
- Reports read/IO errors with platform-specific alert dialogs (macOS) or logging (SDL)
- Reports XML parsing errors with line numbers and source identification for debugging
- Reports interpretation errors with throttling to prevent log spam (max 7 errors)
- Requests parsing abort when error count exceeds threshold
- Maintains source name context for clearer error messages

## External Dependencies
- **Includes:** `<string.h>` (C standard), `cseries.h` (platform abstraction layer), `Logging.h` (logging macros).
- **Inherited:** `XML_Configure` (base class with `DoParse()`, `GetNumInterpretErrors()`, `Buffer`, `BufLen`).
- **External symbols:** Platform-specific APIs (`ExitToShell`, `SimpleAlert`, `ParamText`, `Alert` on macOS; `fprintf`, `exit` on SDL); logging functions (`logAnomaly1`, `logAnomaly`); cseries functions (`csprintf`, `psprintf`).

# Source_Files/XML/XML_DataBlock.h
## File Purpose
Defines `XML_DataBlock`, a class for parsing XML configuration data from in-memory buffers rather than files. Extends `XML_Configure` to enable the Marathon engine to load configuration from memory blocks. Provides error reporting context via a source name field for debugging.

## Core Responsibilities
- Parse XML from character buffers (`ParseData()` entry point)
- Override base-class callbacks to handle data retrieval and error reporting
- Track the source name of XML data for error messaging
- Delegate parsing logic to parent class (`DoParse()`)
- Report read, parse, and interpretation errors with optional source context

## External Dependencies
- **Direct includes:** `XML_Configure.h`
- **Inherited from `XML_Configure`:** `expat.h`, `XML_ElementParser.h`, and expat parser library
- **Defined elsewhere:** `XML_Configure` base class (provides `DoParse()`, `CurrentElement`, parser infrastructure)
- **Standard library:** `<cstddef>` (implicit; `size_t`)

# Source_Files/XML/XML_ElementParser.cpp
## File Purpose
Implements XML element parser infrastructure for the Aleph One game engine. Provides hierarchical element composition, type-safe value parsing from strings, UTF-8 decoding, and error reporting utilities.

## Core Responsibilities
- Construct and manage XML element parser hierarchy with parent-child relationships
- Provide type-safe wrappers for reading numerical (int16/32, uint16/32, float) and boolean values from XML attribute strings
- Implement case-insensitive string matching for XML tags and attributes
- Convert UTF-8 encoded strings to ASCII/MacRoman representation with full parsing state machine
- Manage error states and error message reporting during parsing
- Recursively reset child elements to initial state

## External Dependencies
- **cseries.h**: platform abstractions, type definitions
- **cstypes.h**: int16, uint16, int32, uint32, uint8 types
- **&lt;string.h&gt;**: strlen(), strcpy()
- **&lt;ctype.h&gt;**: toupper()
- **&lt;vector&gt;** (STL): dynamic array for Children
- **unicode_to_mac_roman()**: defined elsewhere; converts Unicode codepoints to MacRoman bytes

# Source_Files/XML/XML_ElementParser.h
## File Purpose
Defines a base class for hierarchical XML element parsing and provides utility functions for parsing numerical, boolean, and string values. Subclasses implement parsing logic for specific XML element types in the Aleph One game engine.

## Core Responsibilities
- Base class (`XML_ElementParser`) for implementing XML element parsers via inheritance
- Template methods for reading and validating typed numerical values (with optional bounds checking)
- Convenience wrappers for common integer, unsigned, float, and boolean value parsing
- ParentΓÇôchild hierarchy management for nested XML elements
- Virtual hooks for element lifecycle (Start/End), attribute handling, string data, and value reset
- Error reporting utilities for missing/invalid attributes, numerical/boolean parsing failures, and out-of-range values
- Global utility functions for case-insensitive string comparison, UTF-8 to ASCII conversion

## External Dependencies
- `<vector>` (STL container for child elements)
- `<stdio.h>` (`sscanf` for numerical parsing)
- `cstypes.h` (platform-specific integer types: `int16`, `uint16`, `int32`, `uint32`)
- `XML_GetBooleanValue()` (defined elsewhere; parses boolean strings)
- `StringsEqual()` (defined elsewhere; case-insensitive comparison)
- `DeUTF8*()` functions (defined elsewhere; UTF-8 conversion utilities)

# Source_Files/XML/XML_LevelScript.cpp
## File Purpose
Manages XML-based level scripts for the Aleph One game engine. Loads script definitions from map files (resource 128), parses XML commands (MML, music, movies, Lua, load screens), and executes them at appropriate game lifecycle points (level start, game end, restoration). Supports pseudo-levels for defaults, restoration, and end-of-game sequences.

## Core Responsibilities
- Load and parse XML level scripts from map file resources
- Execute level-specific MML, music, movie, and Lua scripts
- Manage pseudo-levels (Default, Restore, End) for global configurations
- Search scripts for movie specifications and provide them to playback systems
- Process embedded MML and Lua script chunks from WAD structures
- Provide XML parser infrastructure for script commands and attributes
- Track end-of-game screen configuration

## External Dependencies
- **cseries.h** ΓÇö basic types, macros
- **XML_DataBlock.h** ΓÇö XML parsing base class (LSXML_Loader)
- **XML_ParseTreeRoot.h** ΓÇö RootParser, SetupParseTree(), ResetAllMMLValues()
- **Music.h** ΓÇö Music::instance() singleton, level music management
- **OGL_LoadScreen.h** ΓÇö OGL_LoadScreen::instance() load screen display
- **ColorParser.h** ΓÇö Color_GetParser(), Color_SetArray() for color parsing
- **images.h** ΓÇö get_text_resource_from_scenario() for resource loading
- **lua_script.h** ΓÇö LoadLuaScript() for Lua code execution
- **AStream.h** ΓÇö AIStreamBE binary stream reading
- **FileHandler.h** ΓÇö FileSpecifier, DirectorySpecifier file management
- **map.h** ΓÇö LEVEL_NAME_LENGTH constant for script headers

# Source_Files/XML/XML_LevelScript.h
## File Purpose
Declares the interface for loading, parsing, and executing XML-based level scripts in map files. Manages level-specific MML (Marathon Markup Language) and Lua scripts, handles movie playback specifications for levels and game endings, and provides restoration of default parameter values.

## Core Responsibilities
- Load XML level scripts from map file resources (resource 128)
- Execute level-specific scripts (Pfhortran, MML) when entering a level
- Manage embedded MML and Lua script data (get/set)
- Locate and retrieve movie file specifications for level playback
- Execute end-of-game scripts and restoration scripts
- Provide XML parser for external parsing of default level scripts

## External Dependencies
- **FileHandler.h**: `FileSpecifier` class for file path abstraction
- **XML_ElementParser** (defined elsewhere): XML parsing interface, instantiated by `ExternalDefaultLevelScript_GetParser()`
- **Implicit**: Pfhortran script language runtime (referenced in comments, invoked by RunLevelScript)
- **Implicit**: MML (Marathon Markup Language) parser (invoked by RunLevelScript)
- **Implicit**: Lua script runtime (invoked via embedded Lua data)

---

**Notes**: This header is a key integration point between the map resource layer (FileHandler) and the scripting systems (Pfhortran, MML, Lua, XML). It abstracts level-specific customization in the Aleph One engine, allowing maps to define behavior, aesthetics, and cinematics without engine recompilation.

# Source_Files/XML/XML_Loader_SDL.cpp
## File Purpose
SDL-based implementation of an XML file parser for the Aleph One game engine. Handles reading and parsing XML configuration files from disk, with error reporting at multiple stages (read, parse, interpretation). Supports both single-file and directory-wide parsing.

## Core Responsibilities
- Provide XML file content to the parent parser class via `GetData()`
- Report read errors, parse errors (with line numbers and filenames), and interpretation errors
- Implement file opening, buffering, and cleanup for single XML files
- Scan and parse all valid XML files in a directory recursively
- Limit error output to prevent log spam (max 7 errors shown)
- Filter out non-XML files (.lua scripts, backup files ending in ~)

## External Dependencies
- **Includes:** `cseries.h` (base types, SDL defs), `XML_Loader_SDL.h` (class definition), `FileHandler.h` (file I/O abstractions)
- **STL:** `<vector>`, `<algorithm>` (for `std::sort`)
- **Boost:** `boost::algorithm::string::predicate` (for `ends_with`)
- **Defined elsewhere:** `XML_Configure` base class, `FileSpecifier`, `OpenedFile`, `dir_entry`, `DoParse()`, `GetNumInterpretErrors()`

# Source_Files/XML/XML_Loader_SDL.h
## File Purpose
SDL-based concrete implementation of the XML_Configure parser for loading and parsing XML configuration files. Handles file I/O through FileSpecifier abstraction and provides error reporting with filename context for the Aleph One Marathon engine.

## Core Responsibilities
- Inherit from `XML_Configure` and provide file-based data source for XML parsing
- Load XML file content into memory buffer for expat parser consumption
- Support both single-file and directory-based XML parsing workflows
- Manage file data allocation/deallocation and track file size
- Override error reporting methods to include filename in diagnostic messages
- Implement abort-request handling for parse failure recovery

## External Dependencies
- **Inheritance**: `XML_Configure` (base class providing parse orchestration, element tree, expat integration)
- **Forward Declaration**: `FileSpecifier` (SDL file abstraction; defined elsewhere)
- **Transitive Includes**: `expat.h` (C XML parser), `XML_ElementParser.h` (element handler hierarchy)
- **License**: GNU GPL v2+ (Bungie/Aleph One open-source project)

# Source_Files/XML/XML_MakeRoot.cpp
## File Purpose

Constructs the root of the XML parser tree for Marathon/Aleph One configuration files. Declares global root parser objects and initializes the complete parser hierarchy by attaching subsystem parsers (text strings, interface, player, items, weapons, etc.) as children of the main "marathon" element.

## Core Responsibilities

- Declare and maintain `RootParser` (absolute root) and `MarathonParser` ("marathon" element)
- Build the complete XML parse tree hierarchy via `SetupParseTree()`
- Provide initialization entry point for XML configuration system
- Enable bulk reset of all MML (Marathon Markup Language) configuration values to defaults

## External Dependencies

- **Notable includes:**
  - `XML_ParseTreeRoot.h` (declares `RootParser`, `SetupParseTree()`, `ResetAllMMLValues()`)
  - Subsystem headers (interface.h, player.h, weapons.h, etc.) for subsystem-specific declarations
  - `XML_LevelScript.h` for level script parser

- **External symbols (defined elsewhere):**
  - `TS_GetParser()`, `Interface_GetParser()`, `PlayerName_GetParser()`, `Infravision_GetParser()`, `MotionSensor_GetParser()`, `OverheadMap_GetParser()`, `DynamicLimits_GetParser()`, `AnimatedTextures_GetParser()`, `Player_GetParser()`, `Items_GetParser()`, `ControlPanels_GetParser()`, `Liquids_GetParser()`, `Sounds_GetParser()`, `Platforms_GetParser()`, `Scenery_GetParser()`, `Faders_GetParser()`, `View_GetParser()`, `Landscapes_GetParser()`, `Weapons_GetParser()`, `OpenGL_GetParser()`, `Cheats_GetParser()`, `TextureLoading_GetParser()`, `Keyboard_GetParser()`, `DamageKicks_GetParser()`, `Logging_GetParser()`, `Scenario_GetParser()`, `Theme_GetParser()`, `SW_Texture_Extras_GetParser()`, `Console_GetParser()`, `ExternalDefaultLevelScript_GetParser()` ΓÇö all return `XML_ElementParser*` and are defined in their respective subsystem modules.

# Source_Files/XML/XML_ParseTreeRoot.h
## File Purpose
Header declaring the absolute root element of the XML parser tree and providing initialization/reset routines for the parsing system. Acts as the top-level public interface for XML parsing in the engine.

## Core Responsibilities
- Declare the global `RootParser` as the absolute root element containing all valid XML file roots
- Provide initialization routine to set up the complete parse tree hierarchy
- Provide reset routine to restore all parser values to hardcoded defaults
- Serve as the central entry point for XML document parsing infrastructure

## External Dependencies
- **Includes**: `XML_ElementParser.h` ΓÇö defines the `XML_ElementParser` class (base parser with child management and attribute/string handling)
- **External symbols**: Implementations of both functions (defined elsewhere, presumably XML_ParseTreeRoot.cpp)

# Source_Files/XML/XML_ResourceFork.cpp
## File Purpose
Implementation of XML_ResourceFork class, which parses XML data embedded in MacOS resource forks. Handles resource loading, retrieval, sorting, and both platform-specific (Carbon vs. classic Mac) error reporting with user-facing dialogs.

## Core Responsibilities
- Load individual XML resources from resource forks by type and ID
- Enumerate and sort all resources of a given type, then parse them in ID order
- Report read, parse, and interpretation errors with platform-appropriate dialogs
- Route error messages through parent class (XML_Configure) parsing system
- Provide source name field for debugging (e.g., which file caused the error)
- Handle both Carbon/modern-Mac and classic-Mac API variants

## External Dependencies
- **Includes**: `<algorithm>` (std::sort), `<string.h>`, `cseries.h` (Marathon utilities: csprintf, psprintf, SimpleAlert, ExitToShell), `XML_ResourceFork.h`
- **Base class**: `XML_Configure` (parser state, callbacks)
- **MacOS Resource Manager** (defined elsewhere): `Get1Resource()`, `HLock()`, `HUnlock()`, `ReleaseResource()`, `Count1Resources()`, `Get1IndResource()`, `GetResInfo()`, `SetResLoad()`, `GetHandleSize()`
- **Platform macros**: `TARGET_API_MAC_CARBON` (switches between Carbon and classic Mac dialog APIs)

# Source_Files/XML/XML_ResourceFork.h
## File Purpose
A C++ class wrapper for parsing XML-encoded data stored in MacOS resource forks. Extends the generic `XML_Configure` base class to handle resource fork I/O, allowing Marathon engine configuration files stored as MacOS resources to be parsed into the game engine's configuration system.

## Core Responsibilities
- Parse single XML resources by type and ID
- Parse all XML resources of a given type from a resource fork
- Manage a resource handle and maintain lifecycle of resource data
- Implement platform-specific I/O for MacOS resource fork access
- Report parsing errors (read errors, XML syntax errors, interpretation errors) with source context
- Provide hooks to abort parsing on excessive errors

## External Dependencies
- **Inheritance:** `XML_Configure` (XML parser base; uses expat via static callbacks, `XML_ElementParser` hierarchy)
- **Platform types:** `Handle`, `ResType`, `short` (MacOS/Carbon resource manager API)
- **Includes:** `XML_Configure.h` only

**Notes:**
- Constructor initializes pointers to NULL (safe default)
- `SourceName` is a convenience field for error messages, aiding debugging of misconfigured resource files
- No explicit memory management visible; assumes resource handle lifecycle is managed by caller


