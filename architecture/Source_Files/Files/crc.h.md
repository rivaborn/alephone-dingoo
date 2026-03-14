# Source_Files/Files/crc.h

## File Purpose
Declares CRC (Cyclic Redundancy Check) calculation functions for data integrity verification. Provides interfaces for computing checksums on files (both by FileSpecifier and OpenedFile) and raw data buffers. Part of the Aleph One game engine's file I/O subsystem.

## Core Responsibilities
- Calculate 32-bit CRC for files identified by FileSpecifier reference
- Calculate 32-bit CRC for already-opened files (OpenedFile reference)
- Calculate 32-bit CRC for raw byte buffers
- Calculate 16-bit CRC-CCITT variant for raw byte buffers

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FileSpecifier | class (forward decl.) | File reference abstraction used to identify source files for CRC calculation |
| OpenedFile | class (forward decl.) | Open file handle abstraction for CRC calculation on already-loaded files |

## Global / File-Static State
None.

## Key Functions / Methods

### calculate_crc_for_file
- Signature: `uint32 calculate_crc_for_file(FileSpecifier& File)`
- Purpose: Compute 32-bit CRC checksum for a file specified by FileSpecifier
- Inputs: Reference to FileSpecifier identifying the target file
- Outputs/Return: uint32 CRC value
- Side effects: File I/O (reads file contents)
- Calls: Not inferable from this file (implementation in separate source)
- Notes: Likely opens and reads the file internally

### calculate_crc_for_opened_file
- Signature: `uint32 calculate_crc_for_opened_file(OpenedFile& OFile)`
- Purpose: Compute 32-bit CRC checksum for an already-opened file
- Inputs: Reference to OpenedFile handle
- Outputs/Return: uint32 CRC value
- Side effects: File I/O (reads current file position)
- Calls: Not inferable from this file
- Notes: Avoids re-opening file; operates on in-memory file handle

### calculate_data_crc
- Signature: `uint32 calculate_data_crc(unsigned char *buffer, int32 length)`
- Purpose: Compute 32-bit CRC checksum for raw byte buffer in memory
- Inputs: Byte buffer pointer, buffer length in bytes
- Outputs/Return: uint32 CRC value
- Side effects: None (read-only)
- Calls: Not inferable from this file
- Notes: Core utility for in-memory data; used by file variants

### calculate_data_crc_ccitt
- Signature: `uint16 calculate_data_crc_ccitt(unsigned char *buffer, int32 length)`
- Purpose: Compute 16-bit CRC-CCITT checksum for raw byte buffer (alternate polynomial)
- Inputs: Byte buffer pointer, buffer length in bytes
- Outputs/Return: uint16 CRC value
- Side effects: None (read-only)
- Calls: Not inferable from this file
- Notes: CRC-CCITT variant; smaller output width than standard 32-bit CRC

## Control Flow Notes
Utility/service module for file integrity validation. CRC calculations likely invoked during file loading operations in the engine's resource pipeline (file verification before parsing/use). No initialization or shutdown required.

## External Dependencies
- Forward declarations: `FileSpecifier`, `OpenedFile` (file abstraction classes defined elsewhere)
- Standard type definitions: `uint32`, `uint16`, `int32` (likely from platform headers)
