# Source_Files/CSeries/csstrings.h

## File Purpose

Header file providing string manipulation and encoding utilities for the Aleph One game engine. Bridges legacy Pascal-style strings (Str255) and modern C/C++ strings with platform-specific encoding support (Mac Roman Γåö Unicode/UTF-8).

## Core Responsibilities

- Declare Pascal-style string operations (pstrcpy, pstrncpy, pstrdup)
- Declare C-style string wrappers around resource-based strings (getcstr, getpstr)
- Provide printf-like functions for debug output (dprintf, fdprintf, csprintf, psprintf)
- Support in-place string format conversion (C Γåö Pascal)
- Support character encoding conversions (Mac Roman Γåö Unicode/UTF-8)
- Bridge legacy string types with modern C++ containers (std::string, std::vector)
- Define global temporary string buffer for short-term allocations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `std::string` | type (STL) | Modern C++ string representation |
| `std::vector<std::string>` | type (STL) | Collections of resource strings |

Global temporary buffer `Str255` (via ptemporary macro) provides scratch space for intermediate strings.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `temporary` | `char[256]` | global | General-purpose temporary string buffer |
| `ptemporary` | macro ΓåÆ `Str255*` | global | Cast temporary as Pascal string pointer |

## Key Functions / Methods

### getpstr / getcstr
- Signature: `unsigned char *getpstr(unsigned char *string, short resid, size_t item)` / `char *getcstr(char *string, short resid, size_t item)`
- Purpose: Load string from resource set into caller's buffer
- Inputs: Destination buffer, resource ID, item index
- Outputs/Return: Pointer to destination (caller-provided)
- Side effects: Writes to caller buffer
- Notes: Caller must allocate sufficient buffer space

### pstrcpy / pstrncpy / pstrdup
- Purpose: Pascal string copy (fixed-length) / copy with byte limit / allocate and copy
- Signature: `pstrcpy(dst, src)`, `pstrncpy(dest, source, total_byte_count)`, `pstrdup(source)`
- Outputs/Return: Pointer to destination (caller-allocated or heap-allocated)
- Side effects: pstrdup performs heap allocation
- Notes: pstrncpy includes byte count in first byte of Pascal string

### a1_c2pstr / a1_p2cstr
- Purpose: Convert C string to Pascal in-place (and vice versa)
- Signature: `a1_c2pstr(char* inoutStringBuffer)` / `a1_p2cstr(unsigned char* inoutStringBuffer)`
- Inputs: String buffer to convert (modified in place)
- Outputs/Return: Pointer to converted buffer
- Side effects: Modifies input buffer; Pascal format requires length byte at offset 0

### csprintf / psprintf / dprintf / fdprintf
- Purpose: Formatted output (C/Pascal variants or debug)
- Signature: `csprintf(char *buffer, const char *format, ...)` etc.
- Inputs: Format string + variadic args
- Outputs/Return: Pointer to buffer (csprintf, psprintf) or void (dprintf, fdprintf)
- Side effects: dprintf outputs to console; fdprintf writes to AlephOneDebugLog.txt file
- Notes: PRINTF_STYLE_ARGS enables GCC format checking

### mac_roman_to_unicode / unicode_to_mac_roman
- Purpose: Character and string encoding conversion
- Signature: `uint16 mac_roman_to_unicode(char c)` (character) and 2 string overloads
- Inputs: Single character or string buffer + optional max length
- Outputs/Return: Converted value or modified output buffer
- Side effects: Writes to output buffer (string variants)

### copy_string_to_pstring / copy_string_to_cstring / pstring_to_string
- Purpose: Bridge modern std::string with legacy formats
- Inputs: Source std::string, destination buffer (with optional max length)
- Outputs/Return: Pointer to destination or new std::string
- Side effects: Writes to caller buffer or heap allocation (return case)
- Notes: Default maxlen=255 (Str255 size)

### build_stringvector_from_stringset / mac_roman_to_utf8 / utf8_to_mac_roman
- Purpose: Batch operations (resourceΓåÆvector, encoding conversions on strings)
- Inputs: Resource ID or std::string
- Outputs/Return: std::vector<std::string> or converted std::string
- Side effects: Heap allocation (return values)

## Control Flow Notes

This is a pure interface header (declarations only). Functions are called throughout the engine for:
- **Resource loading**: Early initialization to load UI strings, dialog text, etc.
- **Debug output**: Runtime logging via dprintf/fdprintf
- **String formatting**: Ad-hoc formatting during gameplay and UI updates
- **Encoding**: On Mac platforms for Mac Roman Γåö Unicode conversion

## External Dependencies

- `cstypes.h` ΓÇô cross-platform integer type definitions (int8, uint16, uint32, etc.)
- `<string>`, `<vector>` ΓÇô C++ Standard Library
- Implicit: Mac Toolbox (Str255, SInt types) when compiled on Mac platform
- Implicit: GCC compiler features (format attribute) detected via `__GNUC__`

## Notes

- Code layered with legacy support: Pascal strings from 1991, C++ std::string additions circa 2001
- PRINTF_STYLE_ARGS macro provides compile-time format string validation on GCC/Clang
- `fdprintf` is undocumented here but writes debug output to a file (useful for post-mortem debugging)
- Global `temporary` buffer is a code smellΓÇöcallers should allocate their own buffers where possible
