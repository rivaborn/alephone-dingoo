# Source_Files/CSeries/csfonts.h

## File Purpose
Header file defining font and text styling types for the Aleph One game engine. Provides constants for text style flags (bold, italic, underline, shadow) and a `TextSpec` structure that encapsulates complete font configuration including file paths for different font variants.

## Core Responsibilities
- Define text styling bit flags (normal, bold, italic, underline, shadow)
- Define `TextSpec` struct for storing font metadata and variant paths
- Provide centralized font configuration schema used throughout the engine

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TextSpec` | struct | Encapsulates font specification: ID, styling flags, size, height adjustment, and paths to four font variants (normal, oblique, bold, bold_oblique) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `styleNormal` | const int | global | Text style flag value 0 (no styling) |
| `styleBold` | const int | global | Text style flag value 1 (bold text) |
| `styleItalic` | const int | global | Text style flag value 2 (italic text) |
| `styleUnderline` | const int | global | Text style flag value 4 (underlined text) |
| `styleShadow` | const int | global | Text style flag value 16 (drop shadow effect) |

## Key Functions / Methods
None. This file contains only type and constant definitions.

## Control Flow Notes
Not applicableΓÇöthis is a pure data definition header. No initialization, update, or frame logic.

## External Dependencies
- `cstypes.h`: Platform-specific integer type definitions (`int16`, `uint16`)
- `<string>`: Standard library for `std::string` (font file paths)

## Notes
- `styleOutline` (value 8) is commented out as incompatible with TrueType fonts.
- Font variants are specified via file paths; the struct stores paths for four combinations: normal, oblique (italic), bold, and bold+oblique.
- This header was last modified in SeptΓÇôNov 2001 (per comment crediting Woody Zenfell for `styleOutline`).
