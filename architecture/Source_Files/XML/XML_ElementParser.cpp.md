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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_ElementParser | class | Base class for parseable XML elements; manages child elements and name-based lookup |
| vector\<XML_ElementParser *\> | STL container | Dynamic array of child element parsers |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| InitialErrorString | static char[] | file-static | Default error message on construction |
| UnrecognizedTagString | static char[] | file-static | Error constant for unknown tags |
| AttribsMissingString | static char[] | file-static | Error constant for missing attributes |
| BadNumericalValueString | static char[] | file-static | Error constant for invalid numeric values |
| OutOfRangeString | static char[] | file-static | Error constant for out-of-bounds values |
| BadBooleanValueString | static char[] | file-static | Error constant for invalid boolean values |

## Key Functions / Methods

### XML_ElementParser (constructor)
- Signature: `XML_ElementParser(const char *_Name)`
- Purpose: Initialize a named XML element parser
- Inputs: _Name ΓÇô element name string
- Outputs/Return: none (constructor)
- Side effects: allocates heap memory for Name; initializes ErrorString to InitialErrorString
- Calls: strlen(), operator new, strcpy()
- Notes: Name ownership transferred to parser; must be deallocated by destructor

### ~XML_ElementParser (destructor)
- Signature: `~XML_ElementParser()`
- Purpose: Clean up allocated resources
- Side effects: deallocates Name string
- Calls: operator delete[]
- Notes: does not delete children (caller responsible)

### AddChild
- Signature: `void AddChild(XML_ElementParser *Child)`
- Purpose: Register a child element parser, preventing duplicate child names
- Inputs: Child ΓÇô child parser to register
- Side effects: appends to Children vector if child name not already present
- Calls: Child->GetName(), NameMatch(), Children.push_back()
- Notes: silently ignores duplicate names (returns early without adding)

### FindChild
- Signature: `XML_ElementParser *FindChild(const char *_Name)`
- Purpose: Locate a child element by name
- Inputs: _Name ΓÇô name to search for
- Outputs/Return: pointer to matching child, or NULL if not found
- Calls: NameMatch() for each child
- Notes: linear search through Children vector

### ResetChildrenValues
- Signature: `void ResetChildrenValues()`
- Purpose: Recursively reset all children to initial state
- Side effects: calls ResetValues() and ResetChildrenValues() on each child
- Calls: recursive on all children
- Notes: depth-first traversal

### NameMatch
- Signature: `bool NameMatch(const char *_Name)`
- Purpose: Check if element name matches target (case-insensitive)
- Inputs: _Name ΓÇô name to compare
- Outputs/Return: true if names match
- Calls: StringsEqual()

### StringsEqual
- Signature: `bool StringsEqual(const char *String1, const char *String2, int MaxStrLen = 32)`
- Purpose: Case-insensitive string comparison
- Inputs: two strings, maximum comparison length
- Outputs/Return: true if equal within MaxStrLen characters
- Side effects: none
- Calls: toupper() for each character comparison
- Notes: compares byte-by-byte after converting to uppercase; treats null terminators as equal

### XML_GetBooleanValue
- Signature: `bool XML_GetBooleanValue(const char *String, bool &Value)`
- Purpose: Parse boolean value from string
- Inputs: String ΓÇô one of "1", "t", "true", "0", "f", "false" (case-insensitive)
- Outputs/Return: sets Value to parsed boolean; returns true if valid, false otherwise
- Calls: StringsEqual()
- Notes: six recognized representations; invalid strings return false without modifying Value

### DeUTF8
- Signature: `size_t DeUTF8(const char *InString, size_t InLen, char *OutString, size_t OutMaxLen)`
- Purpose: Decode UTF-8 to ASCII/MacRoman
- Inputs: InString ΓÇô UTF-8 input, InLen ΓÇô input length, OutMaxLen ΓÇô output buffer size
- Outputs/Return: OutString ΓÇô decoded bytes (not null-terminated), returns bytes written
- Side effects: writes to OutString buffer; stops at OutMaxLen or input end
- Calls: unicode_to_mac_roman() for valid codepoints
- Notes: implements complete UTF-8 parsing state machine (1ΓÇô6 byte sequences); replaces invalid sequences with '$'; target range 0x20ΓÇô0xFFFF

### DeUTF8_Pas / DeUTF8_C
- Signature: `DeUTF8_Pas(InString, InLen, OutString, OutMaxLen)` / `DeUTF8_C(...)`
- Purpose: Decode UTF-8 to Pascal or C string format
- Inputs: UTF-8 input and buffer info
- Outputs/Return: OutString ΓÇô formatted string (Pascal: length-prefixed; C: null-terminated)
- Calls: DeUTF8()
- Notes: DeUTF8_Pas asserts length < 256 (Pascal limitation); DeUTF8_C adds null terminator

## Control Flow Notes
Typical XML parsing flow: construct root parser ΓåÆ add child parsers with AddChild() ΓåÆ locate elements via FindChild() ΓåÆ read attribute values via ReadXxxValue methods (many thin wrappers in header) ΓåÆ handle errors via error methods ΓåÆ optionally reset state with ResetChildrenValues(). UTF-8 decoding is independent utility for converting config strings.

## External Dependencies
- **cseries.h**: platform abstractions, type definitions
- **cstypes.h**: int16, uint16, int32, uint32, uint8 types
- **&lt;string.h&gt;**: strlen(), strcpy()
- **&lt;ctype.h&gt;**: toupper()
- **&lt;vector&gt;** (STL): dynamic array for Children
- **unicode_to_mac_roman()**: defined elsewhere; converts Unicode codepoints to MacRoman bytes
