# Source_Files/Misc/Scenario.h

## File Purpose
Manages scenario metadata and compatibility information for the game engine. Implements a singleton that stores scenario name, version, and ID with bounds-checked string fields, and tracks compatible scenario versions. Provides integration with the XML element parser for parsing scenario definition files.

## Core Responsibilities
- Singleton instance management for global scenario metadata access
- Store and retrieve scenario name, version, and ID with length validation
- Track and validate scenario version compatibility
- Provide XML parser integration for parsing scenario elements from configuration files

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Scenario` | class | Singleton managing scenario metadata and compatibility |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Scenario::m_instance` | `Scenario*` | static | Singleton instance pointer |

## Key Functions / Methods

### instance()
- Signature: `static Scenario *instance()`
- Purpose: Return the singleton instance of the Scenario object
- Inputs: None
- Outputs/Return: Pointer to the static `Scenario` instance
- Side effects: Creates instance on first call (lazy initialization pattern)
- Calls: None visible in header
- Notes: Assumes singleton pattern is implemented in `.cpp` file

### SetName / SetVersion / SetID
- Signature: `void SetName(const string name)`, `void SetVersion(const string version)`, `void SetID(const string id)`
- Purpose: Store scenario metadata with automatic length truncation
- Inputs: String values for name (31 chars max), version (7 chars max), ID (23 chars max)
- Outputs/Return: None
- Side effects: Modifies member variables; truncates input strings to specified lengths
- Calls: `std::string` constructor
- Notes: Defensive truncation prevents buffer overflows; max lengths suggest legacy fixed-size format compatibility

### IsCompatible / AddCompatible
- Signature: `bool IsCompatible(const string)`, `void AddCompatible(const string)`
- Purpose: Query and update the list of compatible scenario versions
- Inputs: Version string to check or add
- Outputs/Return: Boolean compatibility status (IsCompatible); void (AddCompatible)
- Side effects: AddCompatible modifies `m_compatibleVersions` vector
- Calls: None visible in header
- Notes: Implementation likely in `.cpp` file; vector stores compatible version strings

### Scenario_GetParser()
- Signature: `XML_ElementParser *Scenario_GetParser()`
- Purpose: Obtain an XML element parser configured to parse `<scenario>` XML elements
- Inputs: None
- Outputs/Return: Pointer to configured `XML_ElementParser` instance
- Side effects: Allocates parser object (caller responsibility for cleanup assumed)
- Calls: None visible in header
- Notes: Implementation in `.cpp` file; entry point for XML parsing integration

## Control Flow Notes
This file defines the scenario metadata layer used during engine initialization/loading phases. The singleton pattern suggests scenarios are loaded once at startup and queried throughout execution. The XML parser integration indicates scenario definitions are read from configuration files (likely during asset loading). Compatibility checking likely occurs when loading maps or mods to validate scenario version requirements.

## External Dependencies
- **Includes**: `XML_ElementParser.h` (XML parsing framework), `<string>`, `<vector>`
- **Defined elsewhere**: `XML_ElementParser` class (provides parser base), `std::string`, `std::vector`
- **License**: GNU GPL v2+ (Bungie Studios / Aleph One developers)
