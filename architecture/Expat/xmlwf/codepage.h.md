# Expat/xmlwf/codepage.h

## File Purpose
Header file declaring the public interface for code page mapping and character conversion utilities. Provides functions to query code page capabilities and convert characters from specific code pages.

## Core Responsibilities
- Declare code page mapping interface to retrieve character mappings for a given code page
- Declare code page conversion interface to convert individual characters from a specified code page
- Establish contracts for code page support in the `xmlwf` tool

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### codepageMap
- Signature: `int codepageMap(int cp, int *map);`
- Purpose: Query the character mapping table for a specified code page.
- Inputs:
  - `cp` (int): Code page identifier
  - `map` (int*): Pointer to output buffer for the mapping table
- Outputs/Return: Integer status code (inferred: 0 for success or error code)
- Side effects: Populates the `map` buffer with code pageΓÇôspecific character mappings
- Calls: Not inferable from this file
- Notes: Caller must allocate sufficient memory for `map`; exact size/format not specified in header

### codepageConvert
- Signature: `int codepageConvert(int cp, const char *p);`
- Purpose: Convert a character or character sequence from a specified code page to an internal representation.
- Inputs:
  - `cp` (int): Code page identifier
  - `p` (const char*): Pointer to character data in the source code page
- Outputs/Return: Integer result (inferred: converted character value or status code)
- Side effects: None visible
- Calls: Not inferable from this file
- Notes: Semantics of multi-byte sequences and return value encoding unclear without implementation

## Control Flow Notes
Not inferable from header. These are utility functions likely called during XML parsing when code page detection/conversion is needed.

## External Dependencies
- Standard C library (inferred from `int` and pointer types; no explicit includes in this header)
