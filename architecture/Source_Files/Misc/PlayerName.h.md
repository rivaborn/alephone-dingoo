# Source_Files/Misc/PlayerName.h

## File Purpose
Header declaring interfaces for retrieving and parsing player names in netgames. Provides a bridge between XML-based configuration (for default player names) and runtime access to the player's name string. Part of the Aleph One engine's configuration system.

## Core Responsibilities
- Declare public accessor function to retrieve the current player name
- Declare factory function to create an XML element parser for "player_name" configuration elements
- Integrate player name configuration into the engine's XML parsing infrastructure

## Key Types / Data Structures
None defined in this file.

## Global / File-Static State
None.

## Key Functions / Methods

### GetPlayerName
- Signature: `unsigned char *GetPlayerName();`
- Purpose: Retrieve the configured player name string
- Inputs: None
- Outputs/Return: Pointer to unsigned char (likely Pascal string; implementation elsewhere)
- Side effects: None inferable from this file
- Calls: Not inferable from this file
- Notes: Return type `unsigned char*` suggests Pascal-style string (length-prefixed) common in older Mac/game engine code

### PlayerName_GetParser
- Signature: `XML_ElementParser *PlayerName_GetParser();`
- Purpose: Factory function to create and return an XML element parser configured for "player_name" XML elements
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser (subclass, implementation elsewhere)
- Side effects: Allocates parser object (caller responsibility to manage lifetime)
- Calls: Not inferable from this file
- Notes: Element name hardcoded as "player_name"; parser handles XML attribute/text parsing and validation

## Control Flow Notes
Likely used during initialization/config-load phase: XML config files are parsed by PlayerName_GetParser() to populate default player name. Runtime code calls GetPlayerName() during netgame setup or in-game UI display. No frame/render-loop involvement inferable.

## External Dependencies
- `XML_ElementParser.h` ΓÇô provides base class for custom element parsers; used to parse structured XML configuration
