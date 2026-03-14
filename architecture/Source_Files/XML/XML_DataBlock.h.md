# Source_Files/XML/XML_DataBlock.h

## File Purpose
Defines `XML_DataBlock`, a class for parsing XML configuration data from in-memory buffers rather than files. Extends `XML_Configure` to enable the Marathon engine to load configuration from memory blocks. Provides error reporting context via a source name field for debugging.

## Core Responsibilities
- Parse XML from character buffers (`ParseData()` entry point)
- Override base-class callbacks to handle data retrieval and error reporting
- Track the source name of XML data for error messaging
- Delegate parsing logic to parent class (`DoParse()`)
- Report read, parse, and interpretation errors with optional source context

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_DataBlock` | class | Extends `XML_Configure`; wraps in-memory XML parsing with buffer management and source tracking |

## Global / File-Static State
None.

## Key Functions / Methods

### ParseData
- Signature: `bool ParseData(char *_Buffer, size_t _BufLen)`
- Purpose: Public entry point to parse XML from a memory buffer
- Inputs: `_Buffer` (pointer to character data), `_BufLen` (byte count of buffer)
- Outputs/Return: Boolean indicating parse success
- Side effects: Sets `Buffer` and `BufLen` members; calls `DoParse()` (inherited), which modifies parsing state and may invoke error callbacks
- Calls: `DoParse()` (inherited from `XML_Configure`)
- Notes: Wrapper that stores buffer pointers, then delegates to inherited parse machinery

### Protected overridable methods
- `GetData()`, `ReportReadError()`, `ReportParseError()`, `ReportInterpretError()`, `RequestAbort()`
  - These are virtual overrides of `XML_Configure` abstract/non-pure methods
  - Called by `DoParse()` during parsing; subclass provides implementation
  - No implementation visible here; rely on base class stubs (which do nothing by default)

### Constructor
- Signature: `XML_DataBlock()`
- Purpose: Initialize instance
- Side effects: Sets `SourceName` to `NULL`, `Buffer` to `NULL`

## Control Flow Notes
Fits into engine **initialization/configuration phase**. Called when the engine needs to read and apply XML-based configuration from a memory buffer (e.g., MOD data, bundled resources). The parsing result drives how engine subsystems are configured.

## External Dependencies
- **Direct includes:** `XML_Configure.h`
- **Inherited from `XML_Configure`:** `expat.h`, `XML_ElementParser.h`, and expat parser library
- **Defined elsewhere:** `XML_Configure` base class (provides `DoParse()`, `CurrentElement`, parser infrastructure)
- **Standard library:** `<cstddef>` (implicit; `size_t`)
