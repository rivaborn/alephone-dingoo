# Expat/xmlwf/xmlfile.h

## File Purpose
Header file that declares the interface for processing XML files. Defines flag constants to control XML parsing behavior (file mapping and external entity handling).

## Core Responsibilities
- Declare `XML_ProcessFile()` function for parsing XML from a file path
- Define flag constants (`XML_MAP_FILE`, `XML_EXTERNAL_ENTITIES`) to configure parsing behavior
- Provide a C API contract for callers in the xmlwf tool

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### XML_ProcessFile
- **Signature:** `int XML_ProcessFile(XML_Parser parser, const XML_Char *filename, unsigned flags)`
- **Purpose:** Parse an XML file and process its contents using the provided parser.
- **Inputs:**
  - `parser`: Initialized XML parser instance (created/configured elsewhere)
  - `filename`: Path to the XML file to process
  - `flags`: Bitmask controlling behavior (see flag constants below)
- **Outputs/Return:** Integer status code (likely 0 for success, non-zero for error; not inferable from this header)
- **Side effects:** Parses and processes XML file; invokes parser callbacks; may perform file I/O and memory allocation
- **Calls:** Not visible in this file (implementation elsewhere)
- **Notes:** Flag constants control memory-mapped file I/O (`XML_MAP_FILE`) and external entity resolution (`XML_EXTERNAL_ENTITIES`)

## Control Flow Notes
This function is part of the **parse/frame** phase of XML processing. Called after parser initialization to process file-based input. Return code likely flows to error handling/reporting upstream.

## External Dependencies
- `XML_Parser`: Opaque parser type (defined elsewhere in expat)
- `XML_Char`: Character type (likely typedef for encoding support, defined elsewhere)
- License headers reference expat library and GPL/MPL dual licensing
