# Expat/xmlwf/codepage.cpp

## File Purpose
Windows codepage conversion utility for the xmlwf XML parser tool. Builds Unicode conversion tables for multibyte codepages and converts individual multibyte sequences to Unicode characters. Platform-specific: full implementation on Windows, stub implementations on other platforms.

## Core Responsibilities
- Build a 256-entry lookup table mapping byte values to Unicode codepoints for a specified codepage
- Convert 2-byte multibyte character sequences to Unicode wide characters
- Identify and mark lead bytes (first byte of multibyte sequences) in codepage mappings
- Validate codepage support via Windows CPINFO structure
- Provide cross-platform interface (functional on Windows, no-ops elsewhere)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CPINFO | struct (Windows API) | Describes codepage properties: max char size, lead byte ranges |

## Global / File-Static State
None.

## Key Functions / Methods

### codepageMap
- Signature: `int codepageMap(int cp, int *map)`
- Purpose: Populate a 256-entry conversion table for a codepage, mapping byte values to Unicode codepoints
- Inputs: `cp` (codepage ID), `map` (pointer to 256-int output array)
- Outputs/Return: `1` on success, `0` on unsupported/invalid codepage
- Side effects: Fills `map[]` array with Unicode values or sentinel markers; calls Windows API
- Calls: `GetCPInfo()`, `MultiByteToWideChar()`
- Notes:
  - Rejects codepages with `MaxCharSize > 2` (3+ byte sequences unsupported)
  - Marks lead bytes as `-2` to distinguish from single-byte characters (marked `-1` initially, then converted)
  - Uses `MB_PRECOMPOSED | MB_ERR_INVALID_CHARS` flags to normalize decomposed forms and reject invalid sequences
  - Windows-only; non-Windows stub returns `0`

### codepageConvert
- Signature: `int codepageConvert(int cp, const char *p)`
- Purpose: Convert a 2-byte multibyte sequence to its Unicode equivalent
- Inputs: `cp` (codepage ID), `p` (pointer to 2-byte sequence)
- Outputs/Return: Unicode codepoint (as `int`) on success, `-1` on conversion failure or unsupported codepage
- Side effects: Calls Windows API
- Calls: `MultiByteToWideChar()`
- Notes:
  - Decodes exactly 2 bytes from `p` into a single wide character
  - Returns `-1` for invalid sequences or on non-Windows platforms
  - Uses same composition/validation flags as `codepageMap()`

## Control Flow Notes
Not part of game engine lifecycle (init/frame/render). Utility library for XML text parsing. Initialization: `codepageMap()` called once per codepage to pre-compute conversion table. Runtime: `codepageConvert()` used to decode individual multibyte characters during XML content parsing.

## External Dependencies
- `<windows.h>` ΓÇö Windows API (GetCPInfo, MultiByteToWideChar); conditional compilation on `WIN32`
- `codepage.h` ΓÇö Header declaring both functions
- **Defined elsewhere**: CPINFO structure, GetCPInfo, MultiByteToWideChar (Windows API)
