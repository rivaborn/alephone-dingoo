# Source_Files/RenderOther/TextStrings.h

## File Purpose
Header file declaring a text-string repository interface that replaces legacy MacOS STR# resources. Provides functions to store, retrieve, and manage strings organized by resource ID and index, supporting both Pascal and C string formats.

## Core Responsibilities
- Store and retrieve strings in Pascal (length-byte prefix) and C (null-terminated) formats
- Manage string collections grouped by resource ID
- Query existence and count of string sets
- Delete individual strings or entire string sets
- Provide XML parsing interface for string data loading

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_ElementParser | class | Parser for loading string data from XML; declared but defined elsewhere |

## Global / File-Static State
None.

## Key Functions / Methods

### TS_PutString
- Signature: `void TS_PutString(short ID, short Index, unsigned char *String)`
- Purpose: Store a Pascal-format string in the repository
- Inputs: Resource ID, index within that set, pointer to Pascal string (length byte + chars + null)
- Outputs/Return: None
- Side effects: Modifies repository; replaces existing string if already present
- Notes: String format is `[length][characters][null]` for dual Pascal/C compatibility

### TS_PutCString
- Signature: `void TS_PutCString(short ID, short Index, const char *String)`
- Purpose: Store a C-style null-terminated string in the repository
- Inputs: Resource ID, index, null-terminated C string
- Outputs/Return: None
- Side effects: Modifies repository (internally converts to Pascal format)

### TS_GetString
- Signature: `unsigned char *TS_GetString(short ID, size_t Index)`
- Purpose: Retrieve a string in Pascal format
- Inputs: Resource ID, index
- Outputs/Return: Pointer to Pascal string; NULL if ID/index invalid
- Side effects: None

### TS_GetCString
- Signature: `char *TS_GetCString(short ID, size_t Index)`
- Purpose: Retrieve a string in C format
- Inputs: Resource ID, index
- Outputs/Return: Pointer to C string; NULL if ID/index invalid
- Side effects: None

### TS_IsPresent, TS_CountStrings, TS_DeleteString, TS_DeleteStringSet, TS_DeleteAllStrings
- Query/delete utilities for string set management; see signatures for details

### TS_GetParser
- Signature: `XML_ElementParser *TS_GetParser()`
- Purpose: Obtain XML parser for loading string sets from XML data
- Outputs/Return: Pointer to parser (do not delete)
- Notes: Parser element name is "stringset"

## Control Flow Notes
This appears to be a utility layer called during engine initialization to load strings (via XML parser) and accessed at runtime during UI/HUD rendering to retrieve localized or data-driven text. Not part of core frame loop.

## External Dependencies
- `XML_ElementParser` class (defined elsewhere; likely in an XML parsing subsystem)
- No standard library includes visible here (implementation likely includes stdio, memory utilities)
