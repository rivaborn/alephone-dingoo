# PBProjects/confpaths.h

## File Purpose
Header file for configuration path definitions in the Aleph One SDL project. Currently contains only a commented-out preprocessor directive, likely representing an unused or disabled configuration path constant.

## Core Responsibilities
- Define package/resource directory paths at compile time
- Provide centralized location for resource path configuration
- Enable conditional compilation of path definitions via preprocessor directives

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PKGDATADIR` | Preprocessor macro (commented out) | Global | Path to application resources (currently disabled) |

## Key Functions / Methods
None.

## Control Flow Notes
Not inferable from this file. No control flow is present.

## External Dependencies
- Standard C preprocessor directives only
- No header includes or external dependencies visible

## Notes
- The macro definition for `PKGDATADIR` points to `"AlephOneSDL.app/Contents/Resources/DataFiles/"`, suggesting this is macOS-specific resource path handling (`.app` bundle structure)
- The commented-out state suggests this configuration is either legacy, superseded by runtime path detection, or disabled during a refactor
- File appears intentionally minimalΓÇölikely a stub or legacy configuration header
