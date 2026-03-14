# Expat/sample/elements.cpp

## File Purpose
Sample application demonstrating the Expat XML parser library. Reads an XML document from a file (selected via cross-platform dialog) and outputs the element tree structure with indentation, attributes, and character data, along with error reporting on parse failures.

## Core Responsibilities
- Create and configure an XML parser with user-defined callbacks
- Display XML element hierarchy with tab indentation based on nesting depth
- Print element attributes prefixed with `$`
- Display text content between elements bracketed with `<<<` and `>>>`
- Handle file I/O with cross-platform open dialog support
- Report parse errors with line numbers
- Clean up parser resources on completion or error

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_Parser` | opaque type | Handle to the Expat parser instance |
| `OpenParameters` | struct (from OpenSave_Interface.h) | Cross-platform file dialog configuration |

## Global / File-Static State
None.

## Key Functions / Methods

### startElement
- **Signature:** `void startElement(void *userData, const char *name, const char **atts)`
- **Purpose:** Callback invoked by parser when an opening tag is encountered.
- **Inputs:** `userData` = pointer to depth counter; `name` = element tag name; `atts` = null-terminated array of attribute name/value pairs
- **Outputs/Return:** None (void); prints to stdout
- **Side effects:** Modifies depth counter, writes indented element name and attributes to stdout
- **Calls:** `putchar()`, `puts()`
- **Notes:** Increments depth after printing. Attributes are iterated as a flat array and printed with `$` prefix. Indentation uses tab characters.

### endElement
- **Signature:** `void endElement(void *userData, const char *name)`
- **Purpose:** Callback invoked by parser when a closing tag is encountered.
- **Inputs:** `userData` = pointer to depth counter; `name` = element tag name
- **Outputs/Return:** None (void)
- **Side effects:** Decrements depth counter
- **Calls:** (none)
- **Notes:** Simple depth management; name parameter is unused.

### CharacterDataHandler
- **Signature:** `void CharacterDataHandler(void *userData, const XML_Char *s, int len)`
- **Purpose:** Callback invoked by parser for text content between elements.
- **Inputs:** `userData` = pointer to depth counter; `s` = character data (not null-terminated); `len` = byte length
- **Outputs/Return:** None (void); prints to stdout
- **Side effects:** Writes indented text content (bracketed with `<<<` and `>>>`) to stdout
- **Calls:** `putchar()`, `printf()`
- **Notes:** Data is not null-terminated; loop uses explicit length. Indentation matches current element depth.

### main
- **Signature:** `int main()`
- **Purpose:** Entry point. Orchestrates file selection, XML parsing, and cleanup.
- **Inputs:** None (command-line arguments unused)
- **Outputs/Return:** 0 on success, -1 on file dialog cancellation or open error, 1 on parse error
- **Side effects:** Displays cross-platform open dialog, opens file, creates parser, reads and parses file in chunks, frees parser, prints status messages to stdout/stderr
- **Calls:** `OpenFile()`, `fopen()`, `fprintf()`, `printf()`, `XML_ParserCreate()`, `XML_SetUserData()`, `XML_SetCharacterDataHandler()`, `XML_SetElementHandler()`, `fread()`, `XML_Parse()`, `XML_ErrorString()`, `XML_GetErrorCode()`, `XML_GetCurrentLineNumber()`, `XML_ParserFree()`, `free()`
- **Notes:** Reads file in `BUFSIZ`-sized chunks; final chunk (when `len < sizeof(buf)`) triggers parser finalization. Error reporting includes line number and error code string. All non-XML output is prefixed with `***`.

## Control Flow Notes
- **Initialization (main):** Dialog selection ΓåÆ file open ΓåÆ parser creation ΓåÆ callback registration
- **Update loop (main):** Reads file chunks in a `do-while` loop, parses each chunk, checks completion flag
- **Callbacks (startElement, endElement, CharacterDataHandler):** Invoked asynchronously by parser during `XML_Parse()` calls
- **Shutdown (main):** Parser freed after loop completes; success message printed

## External Dependencies
- **Includes:** `stdio.h` (standard I/O), `stdlib.h` (memory, exit), `xmlparse.h` (Expat parser), `OpenSave_Interface.h` (cross-platform file dialog)
- **External symbols:** `XML_ParserCreate()`, `XML_SetUserData()`, `XML_SetCharacterDataHandler()`, `XML_SetElementHandler()`, `XML_Parse()`, `XML_ParserFree()`, `XML_ErrorString()`, `XML_GetErrorCode()`, `XML_GetCurrentLineNumber()`, `OpenFile()` (defined elsewhere; user-data callback parameters defined in xmlparse.h)
