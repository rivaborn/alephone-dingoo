# Source_Files/Misc/Scenario.cpp

## File Purpose
Implements the Scenario singleton and XML parsers for handling scenario metadata and compatibility checking in Aleph One. Parses XML scenario elements (name, version, id) and maintains a list of compatible scenario versions for multiplayer compatibility checks.

## Core Responsibilities
- Provide singleton access to the current scenario's metadata (name, version, id)
- Parse XML `<scenario>` elements and their attributes via `XML_ScenarioParser`
- Parse XML `<can_join>` elements containing compatible version strings via `XML_CanJoinParser`
- Store and query compatible scenario versions for compatibility validation
- Supply an XML parser entry point for the engine's XML parsing pipeline

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Scenario` | class | Singleton holding scenario metadata and compatible versions |
| `XML_ScenarioParser` | class | XML element parser for `<scenario>` tags and attributes |
| `XML_CanJoinParser` | class | XML element parser for `<can_join>` tag content |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Scenario::m_instance` | `Scenario*` | static | Singleton instance pointer |
| `ScenarioParser` | `XML_ScenarioParser` | static file-scope | Reusable parser for scenario elements |
| `CanJoinParser` | `XML_CanJoinParser` | static file-scope | Reusable parser for can_join elements |

## Key Functions / Methods

### Scenario::instance()
- **Signature:** `static Scenario *instance()`
- **Purpose:** Lazy-initialize and return the singleton Scenario instance
- **Inputs:** None
- **Outputs/Return:** Pointer to static Scenario instance
- **Side effects:** Allocates `Scenario` on first call; modifies `m_instance`
- **Calls:** Constructor `Scenario()` (via `new` on first call)
- **Notes:** Classic singleton pattern; no thread safety

### Scenario::AddCompatible()
- **Signature:** `void AddCompatible(const string Compatible)`
- **Purpose:** Register a scenario version as compatible with the current scenario
- **Inputs:** `Compatible` ΓÇô a scenario version string (truncated to 23 chars)
- **Outputs/Return:** None
- **Side effects:** Appends to `m_compatibleVersions` vector
- **Calls:** `string()` constructor (truncation)
- **Notes:** Truncates input to 23 characters; no duplicate checking

### Scenario::IsCompatible()
- **Signature:** `bool IsCompatible(const string Compatible)`
- **Purpose:** Check if a given scenario version is compatible with the current scenario
- **Inputs:** `Compatible` ΓÇô scenario version string to validate
- **Outputs/Return:** `true` if compatible, `false` otherwise
- **Side effects:** None
- **Calls:** None (linear search on `m_compatibleVersions`)
- **Notes:** Returns `true` for empty strings or unset IDs; performs direct string equality on each entry; O(n) lookup

### XML_ScenarioParser::HandleAttribute()
- **Signature:** `bool HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Parse XML scenario element attributes (name, version, id)
- **Inputs:** `Tag` ΓÇô attribute name; `Value` ΓÇô attribute value
- **Outputs/Return:** `true` if recognized; `false` + `UnrecognizedTag()` call if not
- **Side effects:** Calls `Scenario::instance()` setters; calls `UnrecognizedTag()` on unknown attribute
- **Calls:** `StringsEqual()` (string comparison), `Scenario::instance()->Set*()` methods
- **Notes:** Recognizes "name", "version", "id" only; case-sensitive comparison

### XML_CanJoinParser::HandleString()
- **Signature:** `bool HandleString(const char *String, int Length)`
- **Purpose:** Parse text content of `<can_join>` elements and register as compatible version
- **Inputs:** `String` ΓÇô character buffer; `Length` ΓÇô byte count
- **Outputs/Return:** `true` on success
- **Side effects:** Calls `Scenario::instance()->AddCompatible()` with substring
- **Calls:** `string()` constructor (create substring), `Scenario::instance()->AddCompatible()`
- **Notes:** Creates temporary string from char buffer and length

### Scenario_GetParser()
- **Signature:** `XML_ElementParser *Scenario_GetParser()`
- **Purpose:** Return the root XML parser for scenario elements; configure parser hierarchy
- **Inputs:** None
- **Outputs/Return:** Pointer to `ScenarioParser` (configured with `CanJoinParser` as child)
- **Side effects:** Calls `ScenarioParser.AddChild(&CanJoinParser)` to register child parser
- **Calls:** `XML_ElementParser::AddChild()`
- **Notes:** Called once during engine initialization to set up the parsing tree; non-const global state modification

## Control Flow Notes
This file is part of the engine's initialization/loading phase. When a scenario XML file is parsed:
1. The engine calls `Scenario_GetParser()` to obtain the root parser
2. XML parser invokes `XML_ScenarioParser::HandleAttribute()` for each attribute on `<scenario>`
3. When `<can_join>` elements are encountered, `XML_CanJoinParser::HandleString()` is called for each version string
4. After loading, game code queries `Scenario::instance()->IsCompatible()` to validate multiplayer compatibility

## External Dependencies
- **Base class:** `XML_ElementParser` (defined elsewhere; provides parsing framework)
- **Standard library:** `<string>`, `<vector>` (STL containers)
- **Utilities:** `StringsEqual()` (string comparison utility, likely from cseries.h)
- **Includes:** `cseries.h` (general Aleph One cross-platform utilities), `Scenario.h` (class definition)
