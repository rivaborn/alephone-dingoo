# Source_Files/CSeries/csstrings.cpp

## File Purpose

String and character encoding utility module for the Aleph One game engine. Provides resource-based string retrieval (Pascal and C strings), formatting functions, and bidirectional character encoding converters (Mac Roman Γåö Unicode Γåö UTF-8) to support legacy Mac string formats and international text.

## Core Responsibilities

- Retrieve strings from resource system (Pascal and C formats)
- Convert between Pascal strings (length-prefixed) and C strings (null-terminated)
- Perform sprintf-like formatting into Pascal and C string buffers
- Build std::vector/std::string from resource string sets
- Convert character encodings: Mac Roman Γåö Unicode, Unicode Γåö UTF-8
- Provide deprecated logging functions (dprintf, fdprintf) that delegate to Logger framework

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `MacRomanUnicodeConverter` | class | Encapsulates Mac Roman Γåö Unicode bidirectional conversion via lookup tables and map; initialized once via static singleton |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `temporary[256]` | char array | file-static | Reusable temporary string buffer |
| `mac_roman_to_unicode_table[256]` | uint16 array | class-static | 256-entry lookup table mapping Mac Roman bytes (0x00ΓÇô0xFF) to Unicode codepoints |
| `unicode_to_mac_roman` | std::map<uint16, char> | class-static | Reverse mapping for non-ASCII Unicode codepoints to Mac Roman bytes |
| `initialized` | bool | class-static | One-time initialization flag for unicode_to_mac_roman population |
| `macRomanUnicodeConverter` | MacRomanUnicodeConverter | file-static | Singleton instance; initialized on first use |

## Key Functions / Methods

### countstr
- Signature: `size_t countstr(short resid)`
- Purpose: Count number of strings in a resource set
- Inputs: `resid` ΓÇô resource ID identifying a string set
- Outputs/Return: Count of contiguous strings from index 0
- Side effects: None (read-only query)
- Calls: `TS_CountStrings(resid)` (defined elsewhere)

### getpstr
- Signature: `unsigned char *getpstr(unsigned char *string, short resid, size_t item)`
- Purpose: Retrieve a Pascal string from resources and copy into caller's buffer
- Inputs: `string` ΓÇô output buffer; `resid` ΓÇô resource ID; `item` ΓÇô index in set
- Outputs/Return: Pointer to `string` buffer (or empty string if not found)
- Side effects: Writes to caller's buffer; buffer must be ΓëÑ256 bytes
- Calls: `TS_GetString(resid, item)`, `memcpy()`
- Notes: Pascal string format: [length_byte][chars]; empty result on lookup failure

### getcstr
- Signature: `char *getcstr(char *string, short resid, size_t item)`
- Purpose: Retrieve a C string from resources and copy into caller's buffer
- Inputs: `string` ΓÇô output buffer; `resid` ΓÇô resource ID; `item` ΓÇô index in set
- Outputs/Return: Pointer to `string` buffer (or null-terminated empty string if not found)
- Side effects: Writes to caller's buffer
- Calls: `TS_GetString(resid, item)`, `memcpy()`
- Notes: Extracts C string by skipping Pascal length byte; resource is Pascal-prefixed

### build_stringvector_from_stringset
- Signature: `const vector<string> build_stringvector_from_stringset(int resid)`
- Purpose: Create a std::vector<string> from all strings in a resource set
- Inputs: `resid` ΓÇô resource ID
- Outputs/Return: Vector of std::strings built from contiguous resource entries
- Side effects: Dynamic allocation for vector and strings
- Calls: `TS_GetCString(resid, index)` in loop until NULL
- Notes: Terminates at first NULL (gap in indices)

### csprintf / psprintf
- Signature: `char *csprintf(char *buffer, const char *format, ...)`; `unsigned char *psprintf(unsigned char *buffer, const char *format, ...)`
- Purpose: Format variadic args into C or Pascal string buffers
- Inputs: `buffer` ΓÇô output; `format` ΓÇô printf-style format string; `...` ΓÇô variadic args
- Outputs/Return: Pointer to buffer
- Side effects: Writes formatted string to buffer; psprintf sets Pascal length byte at `buffer[0]`
- Calls: `va_start`, `vsprintf`, `va_end`, `strlen`
- Notes: No bounds checking; caller responsible for buffer size

### dprintf / fdprintf
- Signature: `void dprintf(const char* format, ...)`; `void fdprintf(const char* format, ...)`
- Purpose: Deprecated logging functions; delegate to Logger framework
- Inputs: `format` ΓÇô printf-style format; `...` ΓÇô variadic args
- Outputs/Return: None
- Side effects: Sends log message to current Logger at logAnomalyLevel
- Calls: `GetCurrentLogger()->logMessageV(logDomain, logAnomalyLevel, "unknown", 0, format, list)`
- Notes: Both functions now identical; users should migrate to log* macros in Logging.h

### Copy/convert helpers
**a1_p2cstr / a1_c2pstr** ΓÇô In-place buffer conversions (memmove length byte); **pstrncpy** ΓÇô Safe bounded Pascal copy with null-padding; **pstrdup** ΓÇô Malloc'd Pascal string duplicate; **copy_string_to_pstring / copy_string_to_cstring** ΓÇô Convert std::string to Pascal/C form; **pstring_to_string** ΓÇô Convert Pascal string to std::string.

### Character encoding
**mac_roman_to_unicode** (3 overloads) ΓÇô Convert Mac Roman char or string to Unicode codepoints; **unicode_to_mac_roman** ΓÇô Reverse single-char conversion; **mac_roman_to_utf8 / utf8_to_mac_roman** ΓÇô Full-string encoding conversion; **unicode_to_utf8 / utf8_to_unicode** (static helpers) ΓÇô Encode/decode individual Unicode codepoints to/from UTF-8 byte sequences. All use static singleton `macRomanUnicodeConverter` for lookups.

## Control Flow Notes

Library module; no frame-update pattern. String and encoding functions are called on-demand throughout engine lifetime: resource strings fetched early and during UI rendering; character conversions triggered by text input/output or file I/O. Logging functions (dprintf/fdprintf) are deprecated facades over the Logger framework.

## External Dependencies

- `<stdio.h>`, `<stdarg.h>`, `<string.h>` ΓÇô Standard C library
- `<map>`, `<string>`, `<vector>` ΓÇô STL containers
- `TextStrings.h` ΓÇô Resource string retrieval (TS_GetString, TS_GetCString, TS_CountStrings)
- `Logging.h` ΓÇô Logger framework (GetCurrentLogger, logMessageV, logDomain, logAnomalyLevel)
- `cstypes.h` ΓÇô Type definitions (uint16)
- Conditional: `<Carbon/Carbon.h>` (Mac Carbon target only)
