# Source_Files/XML/XML_ResourceFork.cpp

## File Purpose
Implementation of XML_ResourceFork class, which parses XML data embedded in MacOS resource forks. Handles resource loading, retrieval, sorting, and both platform-specific (Carbon vs. classic Mac) error reporting with user-facing dialogs.

## Core Responsibilities
- Load individual XML resources from resource forks by type and ID
- Enumerate and sort all resources of a given type, then parse them in ID order
- Report read, parse, and interpretation errors with platform-appropriate dialogs
- Route error messages through parent class (XML_Configure) parsing system
- Provide source name field for debugging (e.g., which file caused the error)
- Handle both Carbon/modern-Mac and classic-Mac API variants

## Key Types / Data Structures
None defined locally (class defined in header).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| FatalErrorAlert | const short | global | Dialog ID (128) for fatal resource/parse errors |
| NonFatalErrorAlert | const short | global | Dialog ID (129) for non-fatal interpretation errors |
| MaxErrorsToShow | const int | global | Threshold (7) for suppressing further error dialogs and aborting parse |

## Key Functions / Methods

### GetData
- Signature: `bool GetData()`
- Purpose: Dereference the resource handle and populate Buffer/BufLen for XML parser
- Inputs: None (uses member `ResourceHandle`)
- Outputs/Return: `true` if handle is valid, `false` otherwise
- Side effects: Sets `Buffer`, `BufLen`, `LastOne` members; no I/O
- Calls: `GetHandleSize(ResourceHandle)`
- Notes: Called by inherited `DoParse()` to fetch data chunk-by-chunk; `LastOne=true` signals parser this is final chunk

### ReportReadError
- Signature: `void ReportReadError()`
- Purpose: Display a fatal error dialog when resource fork cannot be read
- Inputs: Uses member `SourceName` (if available, else defaults to "[]")
- Outputs/Return: None (calls `ExitToShell()` to terminate)
- Side effects: Displays platform-specific alert dialog; terminates process
- Calls: `csprintf()` / `psprintf()`, `SimpleAlert()` / `Alert()`, `ExitToShell()`
- Notes: Platform-aware via `TARGET_API_MAC_CARBON` macro

### ReportParseError
- Signature: `void ReportParseError(const char *ErrorString, int LineNumber)`
- Purpose: Report XML parsing errors (malformed XML, invalid syntax) with line number
- Inputs: Error message string, line number in resource
- Outputs/Return: None (exits)
- Side effects: Displays fatal alert; calls `ExitToShell()`
- Calls: `csprintf()` / `psprintf()`, `SimpleAlert()` / `Alert()`, `ExitToShell()`
- Notes: Inherited callback from XML_Configure base class

### ReportInterpretError
- Signature: `void ReportInterpretError(const char *ErrorString)`
- Purpose: Report non-fatal semantic interpretation errors (e.g., invalid config values)
- Inputs: Error message string
- Outputs/Return: None
- Side effects: Displays non-fatal alert (up to `MaxErrorsToShow` times); suppresses further dialogs if threshold exceeded
- Calls: `GetNumInterpretErrors()`, `SimpleAlert()` / `Alert()`, `CopyCStringToPascal()`
- Notes: Does not exit; parent class may eventually call `RequestAbort()`

### RequestAbort
- Signature: `bool RequestAbort()`
- Purpose: Signal parser whether to halt if too many interpretation errors accumulated
- Inputs: None
- Outputs/Return: `true` if error count ΓëÑ `MaxErrorsToShow`, else `false`
- Side effects: None
- Calls: `GetNumInterpretErrors()`
- Notes: Inherited callback; prevents dialog flooding

### ParseResource
- Signature: `bool ParseResource(ResType Type, short ID)`
- Purpose: Load, lock, and parse a single resource by type and ID
- Inputs: MacOS resource type (4-char code), resource ID
- Outputs/Return: `true` if resource found and parsed successfully, `false` if not found
- Side effects: Locks/unlocks handle; calls `DoParse()` (inherited); exits on parse error
- Calls: `Get1Resource()`, `HLock()`, `DoParse()`, `HUnlock()`, `ReleaseResource()`, `ExitToShell()`
- Notes: On parse failure, shows alert and terminates; invoked once per resource by `ParseResourceSet()`

### ParseResourceSet
- Signature: `bool ParseResourceSet(ResType Type)`
- Purpose: Enumerate all resources of a type, sort by ID, and parse each
- Inputs: Resource type code
- Outputs/Return: `true` if at least one resource was found, `false` if none exist
- Side effects: Allocates/deallocates ID array; calls `ParseResource()` for each ID
- Calls: `Count1Resources()`, `SetResLoad()`, `Get1IndResource()`, `GetResInfo()`, `ReleaseResource()`, `std::sort()`, `ParseResource()`
- Notes: **Critical detail**: Disables resource loading (`SetResLoad(false)`) during enumeration to avoid locking all resources in memory; sorting ensures deterministic parse order (resources read in ascending ID order, not file order)

## Control Flow Notes
- `ParseResourceSet()` is the typical entry point; loads all resources of a type in sorted ID order
- Each call to `ParseResourceSet()` ΓåÆ `ParseResource()` ΓåÆ inherited `DoParse()` ΓåÆ `GetData()` (pulling from resource fork)
- Errors at any stage (read, parse, interpret) trigger platform-specific dialogs and may exit
- Interpretation errors are non-fatal and capped at 7 before parse abort

## External Dependencies
- **Includes**: `<algorithm>` (std::sort), `<string.h>`, `cseries.h` (Marathon utilities: csprintf, psprintf, SimpleAlert, ExitToShell), `XML_ResourceFork.h`
- **Base class**: `XML_Configure` (parser state, callbacks)
- **MacOS Resource Manager** (defined elsewhere): `Get1Resource()`, `HLock()`, `HUnlock()`, `ReleaseResource()`, `Count1Resources()`, `Get1IndResource()`, `GetResInfo()`, `SetResLoad()`, `GetHandleSize()`
- **Platform macros**: `TARGET_API_MAC_CARBON` (switches between Carbon and classic Mac dialog APIs)
