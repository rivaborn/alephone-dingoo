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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class (from FileHandler.h) | File/path specification for movie files and scripts |
| `XML_ElementParser` | class (defined elsewhere) | XML parsing interface for script elements |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `EndScreenIndex` | short | extern | Fake level index used for end-of-game screen selection |
| `NumEndScreens` | short | extern | Number of sequential end screens to display (resource IDs increment) |

## Key Functions / Methods

### LoadLevelScripts
- Signature: `void LoadLevelScripts(FileSpecifier& MapFile)`
- Purpose: Load all level scripts from resource 128 in a map file (or platform equivalent)
- Inputs: Map file specifier
- Outputs/Return: None
- Side effects: Populates internal script storage from map resources
- Calls: (Not visible in header)
- Notes: Called once at initialization to load all map-level scripts

### RunLevelScript
- Signature: `void RunLevelScript(int LevelIndex)`
- Purpose: Execute the script for a specific level, including Pfhortran and level-specific MML
- Inputs: Zero-based level index
- Outputs/Return: None
- Side effects: Modifies game state per script execution
- Calls: (Not visible in header; invokes Pfhortran and MML parsers)
- Notes: Called when transitioning into a level

### RunScriptChunks
- Signature: `void RunScriptChunks()`
- Purpose: Execute queued or pending script chunks (timing/deferred execution)
- Inputs: None
- Outputs/Return: None
- Side effects: Processes accumulated script commands
- Notes: Frame-by-frame execution; called from main loop

### FindLevelMovie
- Signature: `void FindLevelMovie(short index)`
- Purpose: Locate and cache the movie file for a given level and the end-game movie
- Inputs: Level index
- Outputs/Return: None (stores result internally)
- Side effects: Caches level and end movie file specifications
- Notes: Searches script data; called before level start

### GetLevelMovie
- Signature: `FileSpecifier *GetLevelMovie(float& PlaybackSize)`
- Purpose: Retrieve the movie file to play for current level, with optional playback size override
- Inputs: Playback size reference (in/out: defaults if not specified in script)
- Outputs/Return: Pointer to FileSpecifier, or NULL if no movie defined
- Side effects: May modify PlaybackSize parameter
- Notes: Returns NULL when no movie is configured

### SetMMLS / GetMMLS
- Signature: `void SetMMLS(uint8* data, size_t length)` / `uint8* GetMMLS(size_t& length)`
- Purpose: Embed/retrieve MML (Marathon Markup Language) script data from a level
- Inputs (Set): Pointer to MML data, byte length
- Outputs (Get): Byte length reference (out)
- Outputs/Return (Get): Pointer to MML data
- Side effects: Stores/retrieves embedded script data
- Notes: Used for level-specific Marathon Markup Language customization

### SetLUAS / GetLUAS
- Signature: `void SetLUAS(uint8* data, size_t length)` / `uint8* GetLUAS(size_t& length)`
- Purpose: Embed/retrieve Lua script data from a level
- Inputs (Set): Pointer to Lua data, byte length
- Outputs (Get): Byte length reference (out)
- Outputs/Return (Get): Pointer to Lua data
- Side effects: Stores/retrieves embedded script data
- Notes: Used for level-specific Lua scripting

### RunEndScript
- Signature: `void RunEndScript()`
- Purpose: Execute script designated for game ending (cinematics, credits, etc.)
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies game state per end-script
- Notes: Called at game completion

### RunRestorationScript
- Signature: `void RunRestorationScript()`
- Purpose: Restore default parameter values by executing a restoration script
- Inputs: None
- Outputs/Return: None
- Side effects: Resets game parameters to defaults (counteracts MML changes across multiple levels)
- Notes: Useful because MML is applied at multiple places; simplifies reset logic

### ResetLevelScript
- Signature: `void ResetLevelScript()`
- Purpose: Reset level script state (clear any cached/active script data)
- Inputs: None
- Outputs/Return: None
- Side effects: Clears internal script buffers/caches
- Notes: Called between levels or on level restart

### ExternalDefaultLevelScript_GetParser
- Signature: `XML_ElementParser *ExternalDefaultLevelScript_GetParser()`
- Purpose: Retrieve the XML parser for external/default level script definitions
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser
- Side effects: None (returns factory/reference to parser)
- Notes: Used by external code to parse standard XML script format

## Control Flow Notes
- **Initialization**: `LoadLevelScripts()` called once at engine startup to load all map-resident scripts
- **Level Entry**: `FindLevelMovie()` then `RunLevelScript()` called when entering a new level
- **Frame Updates**: `RunScriptChunks()` called per frame to process deferred script commands
- **Movie Playback**: `GetLevelMovie()` queried before showing level intro/outro movies
- **Game End**: `RunEndScript()` called after final level completion
- **Parameter Reset**: `RunRestorationScript()` called to undo per-level MML customizations

## External Dependencies
- **FileHandler.h**: `FileSpecifier` class for file path abstraction
- **XML_ElementParser** (defined elsewhere): XML parsing interface, instantiated by `ExternalDefaultLevelScript_GetParser()`
- **Implicit**: Pfhortran script language runtime (referenced in comments, invoked by RunLevelScript)
- **Implicit**: MML (Marathon Markup Language) parser (invoked by RunLevelScript)
- **Implicit**: Lua script runtime (invoked via embedded Lua data)

---

**Notes**: This header is a key integration point between the map resource layer (FileHandler) and the scripting systems (Pfhortran, MML, Lua, XML). It abstracts level-specific customization in the Aleph One engine, allowing maps to define behavior, aesthetics, and cinematics without engine recompilation.
