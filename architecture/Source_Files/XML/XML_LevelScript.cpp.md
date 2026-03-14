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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| LevelScriptCommand | struct | Individual command (MML, Music, Movie, Lua, LoadScreen) with resource ID, file spec, and display properties |
| LevelScriptHeader | struct | Container for commands belonging to one level, with music random-order flag |
| XML_LSCommandParser | class | Parser for individual command elements; accumulates attributes into LevelScriptCommand |
| XML_RandomOrderParser | class | Parser for music playback order attribute |
| XML_GeneralLevelScriptParser | class | Base parser that manages SetLevel() and script lookup/creation |
| XML_LevelScriptParser | class | Parser for regular level scripts with index attribute |
| XML_SpecialLevelScriptParser | class | Parser for pseudo-levels (default, restore, end) |
| XML_EndScreenParser | class | Parser for end-screen configuration |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| LevelScripts | vector\<LevelScriptHeader\> | static | All loaded level scripts indexed by level |
| CurrScriptPtr | LevelScriptHeader* | static | Current script being processed |
| MapParentDir | DirectorySpecifier | static | Directory containing map and related files |
| MovieFile | FileSpecifier | static | Current level's movie file |
| MovieFileExists | bool | static | Whether movie file exists and is valid |
| MovieSize | float | static | Playback size for movie |
| mmls_chunk, luas_chunk | vector\<uint8\> | static | Embedded MML and Lua script binary data |
| EndScreenIndex, NumEndScreens | short | extern | End-of-game screen configuration |
| LuaFound | bool | static | Flag indicating Lua script was successfully loaded |
| Various parser instances | XML_*Parser | static | MMLParser, MusicParser, MovieParser, LuaParser, LoadScreenParser, RandomOrderParser, etc. |

## Key Functions / Methods

### LoadLevelScripts
- **Signature:** `void LoadLevelScripts(FileSpecifier& MapFile)`
- **Purpose:** Load level script definitions from a map file's TEXT resource 128
- **Inputs:** FileSpecifier reference to map file
- **Outputs:** None (modifies global state)
- **Side effects:** Clears LevelScripts vector (except on first call), initializes XML parsing tree, loads resource data
- **Calls:** MapFile.ToDirectory(), SetupLSParseTree(), get_text_resource_from_scenario(), LSXML_Loader.ParseData()
- **Notes:** Uses FirstTime flag to avoid clearing external scripts on initial load

### RunLevelScript
- **Signature:** `void RunLevelScript(int LevelIndex)`
- **Purpose:** Execute default and level-specific scripts for a game level
- **Inputs:** Level index (integer)
- **Outputs:** None
- **Side effects:** Fades music, resets level music, runs MML/music/Lua scripts
- **Calls:** ResetLevelScript(), GeneralRunScript(), Music::instance()->SeedLevelMusic()
- **Notes:** Always runs Default script first, then level-specific

### RunScriptChunks
- **Signature:** `void RunScriptChunks()`
- **Purpose:** Process embedded MML and Lua binary script chunks from WAD structures
- **Inputs:** None (reads mmls_chunk and luas_chunk globals)
- **Outputs:** None
- **Side effects:** Parses embedded scripts, loads Lua code
- **Calls:** AIStreamBE constructor/read(), LSXML_Loader.ParseData(), LoadLuaScript()
- **Notes:** Identical loop structure for MML and Lua; validates offsets against chunk sizes

### GeneralRunScript
- **Signature:** `void GeneralRunScript(int LevelIndex)`
- **Purpose:** Find and execute all commands in a script for given level or pseudo-level
- **Inputs:** Level index or pseudo-level constant (Restore, Default, End)
- **Outputs:** None
- **Side effects:** Sets music random order, loads resources, executes all command types
- **Calls:** Music::instance()->LevelMusicRandom(), get_text_resource_from_scenario(), LSXML_Loader.ParseData(), Music::instance()->PushBackLevelMusic(), OGL_LoadScreen::instance()->Set()
- **Notes:** Searches LevelScripts vector; returns early if script not found; handles MML, Lua, Music, LoadScreen, Movie command types

### FindMovieInScript / FindLevelMovie
- **Signature:** `void FindMovieInScript(int LevelIndex)`, `void FindLevelMovie(short index)`
- **Purpose:** Locate movie specifications in scripts (helper and wrapper)
- **Inputs:** Level index
- **Outputs:** None (updates static MovieFile, MovieFileExists, MovieSize)
- **Side effects:** Sets global movie state
- **Calls:** MovieFile.SetNameWithPath(), FindMovieInScript()
- **Notes:** FindLevelMovie resets state and searches both Default and specific scripts

### GetLevelMovie
- **Signature:** `FileSpecifier *GetLevelMovie(float& Size)`
- **Purpose:** Retrieve prepared movie file and playback size
- **Inputs:** Reference to size variable
- **Outputs:** Pointer to MovieFile or NULL if no movie exists
- **Side effects:** May update Size parameter if MovieSize >= 0
- **Calls:** None
- **Notes:** Simple accessor; Size parameter modified only if MovieSize is non-negative

### Set/GetMMLS, Set/GetLUAS
- **Signature:** `void SetMMLS(uint8* data, size_t length)`, etc.
- **Purpose:** Store and retrieve embedded MML/Lua script chunks
- **Inputs:** Data pointer and length (or length reference for getters)
- **Outputs:** Pointer to chunk data or length
- **Side effects:** Copies data to/from global vectors
- **Calls:** vector::resize(), memcpy()
- **Notes:** Clear vectors if length is 0; GetMMLS/GetLUAS return NULL if empty

## Control Flow Notes
This file participates in the game lifecycle:
- **Initialization:** LoadLevelScripts() called when loading map file to parse XML definitions
- **Per-level:** RunLevelScript() called at level start (runs Default + level-specific scripts)
- **Embedded scripts:** RunScriptChunks() processes binary script chunks from WAD
- **End-of-game:** RunEndScript() runs End pseudo-level script
- **Restoration:** RunRestorationScript() restores saved parameter defaults
- **Movie retrieval:** FindLevelMovie() and GetLevelMovie() called during movie playback

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
