# Source_Files/Misc/PlayerName.cpp

## File Purpose
Manages the player's name in netgames through static storage and XML configuration parsing. Provides a getter function and XML parser to load/initialize the player name with a default fallback ("Marathon Player").

## Core Responsibilities
- Store player name as a static Pascal string (length prefix + data)
- Provide getter access to the stored player name
- Parse `<player_name>` XML elements from configuration files
- Convert UTF-8 text to Pascal string format
- Initialize with a hardcoded default name at parser creation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_SimpleStringParser` | class | XML element parser for `<player_name>` tags; inherits from `XML_ElementParser` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PlayerName` | `unsigned char[256]` | static | Stores player name as Pascal string (byte 0 = length, bytes 1ΓÇô255 = data) |
| `PlayerNameParser` | `XML_SimpleStringParser` | static | Singleton parser instance for XML player_name elements |

## Key Functions / Methods

### GetPlayerName
- Signature: `unsigned char *GetPlayerName()`
- Purpose: Retrieve pointer to the stored player name
- Inputs: None
- Outputs/Return: Pointer to static `PlayerName` array
- Side effects: None
- Calls: None
- Notes: Caller receives direct pointer to static Pascal string buffer; no copying

### XML_SimpleStringParser::HandleString
- Signature: `bool XML_SimpleStringParser::HandleString(const char *String, int Length)`
- Purpose: Callback invoked by XML parser when text content is encountered in `<player_name>` element
- Inputs: `String` (UTF-8 text), `Length` (byte count)
- Outputs/Return: `true` (always succeeds)
- Side effects: Modifies static `PlayerName` array via `DeUTF8_Pas()` call
- Calls: `DeUTF8_Pas()` (defined elsewhere; converts UTF-8 to Pascal string)
- Notes: Overwrites any previous name; truncates/pads to 255 characters max

### PlayerName_GetParser
- Signature: `XML_ElementParser *PlayerName_GetParser()`
- Purpose: Initialize the parser and set default player name; called during engine startup
- Inputs: None
- Outputs/Return: Pointer to static `PlayerNameParser` instance
- Side effects: Initializes `PlayerName` array with "Marathon Player" in Pascal string format
- Calls: `strlen()`, `memcpy()`
- Notes: Must be called before XML parsing begins; assertion checks length Γëñ 255 bytes

## Control Flow Notes
**Initialization phase:** `PlayerName_GetParser()` is called at startup to install the XML parser and set the default name. If no XML configuration is present, the default persists.

**Configuration phase:** When XML is parsed, any `<player_name>` element triggers `HandleString()`, which updates the global name.

**Runtime access:** Game code calls `GetPlayerName()` when needing to display or transmit the player's name in netgames.

## External Dependencies
- **`DeUTF8_Pas()`** ΓÇô UTF-8 to Pascal string converter (defined elsewhere; likely in `csstrings.h`)
- **`XML_ElementParser`** ΓÇô Base class for XML element handlers (`XML_ElementParser.h`)
- **Standard C library** ΓÇô `strlen()`, `memcpy()` from `<string.h>`
