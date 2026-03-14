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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_Loader_SDL` | class | Concrete SDL-based XML file loader; inherits from `XML_Configure` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data` | `char*` | member | Dynamically allocated buffer holding loaded XML file content |
| `data_size` | `int32` | member | Size in bytes of the loaded data buffer |
| `FileName` | `char[256]` | member | Filename string for error message context; initialized to null string |

## Key Functions / Methods

### ParseFile
- Signature: `bool ParseFile(FileSpecifier &file)`
- Purpose: Parse a single XML file specified by FileSpecifier
- Inputs: FileSpecifier reference indicating which file to load
- Outputs/Return: Boolean success indicator
- Side effects: Allocates `data` buffer, populates `data_size`, sets `FileName`
- Calls: (Implementation not visible in header; likely calls inherited `DoParse()`)
- Notes: Caller responsible for passing valid FileSpecifier

### ParseDirectory
- Signature: `bool ParseDirectory(FileSpecifier &dir)`
- Purpose: Parse XML configuration from files within a directory
- Inputs: FileSpecifier reference to target directory
- Outputs/Return: Boolean success indicator
- Side effects: Multiple allocations/deallocations as directory files are processed sequentially
- Calls: (Implementation not visible; likely iterates directory and calls GetData for each file)
- Notes: Supports plugin/mod directory loading pattern common in Marathon-based games

### GetData (virtual override)
- Signature: `virtual bool GetData()`
- Purpose: Provide next chunk of XML data to parser; called repeatedly by inherited `DoParse()`
- Outputs/Return: Boolean indicating if data was successfully read (false signals end of input)
- Side effects: Populates `Buffer`, `BufLen`, `LastOne` members inherited from XML_Configure
- Notes: Contract inherited from XML_Configure; implementation reads from `data` buffer

### Error Reporting Overrides
- **ReportReadError**: Invoked when file I/O fails
- **ReportParseError**: Invoked on XML syntax errors; includes line number
- **ReportInterpretError**: Invoked on semantic/configuration errors
- **RequestAbort**: Allows implementation to signal early parse termination
- Notes: All include `FileName` context in error output (added 2001 by Loren Petrich)

## Control Flow Notes
This class is a data source provider within the larger XML configuration pipeline:
1. **Initialization**: Constructor zeroes FileName, nullifies data pointer
2. **File Load**: `ParseFile()` or `ParseDirectory()` called by engine init
3. **Parsing Loop**: Inherited `DoParse()` repeatedly calls `GetData()` and feeds chunks to expat parser
4. **Cleanup**: Destructor deallocates data buffer on object destruction
5. Does **not** handle XML element interpretationΓÇöthat is delegated to `XML_ElementParser` subtree managed by `XML_Configure`

## External Dependencies
- **Inheritance**: `XML_Configure` (base class providing parse orchestration, element tree, expat integration)
- **Forward Declaration**: `FileSpecifier` (SDL file abstraction; defined elsewhere)
- **Transitive Includes**: `expat.h` (C XML parser), `XML_ElementParser.h` (element handler hierarchy)
- **License**: GNU GPL v2+ (Bungie/Aleph One open-source project)
