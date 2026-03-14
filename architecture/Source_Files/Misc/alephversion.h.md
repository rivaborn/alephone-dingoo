# Source_Files/Misc/alephversion.h

## File Purpose
Version and platform metadata header for Aleph One game engine. Defines version strings, dates, and platform-specific identifiers that are compiled into the engine. Acts as a single source of truth for release versioning across the codebase.

## Core Responsibilities
- Define release version number and display format
- Maintain date-based version identifiers
- Detect and encode platform at compile time (Windows, macOS, Linux, etc.)
- Map detected platform to update-system identifier
- Construct composite version string for display/logging

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| A1_DISPLAY_VERSION | Macro | Global | Semantic version string ("0.22.1") |
| A1_DISPLAY_DATE_VERSION | Macro | Global | Human-readable release date ("2009-10-15") |
| A1_DATE_VERSION | Macro | Global | Numeric date format for version tracking ("20091015") |
| A1_DISPLAY_PLATFORM | Macro | Global (conditional) | Platform name for UI/logs (e.g., "Windows", "Mac OS X") |
| A1_UPDATE_PLATFORM | Macro | Global (conditional) | Platform identifier for update system (e.g., "windows", "macosx", "source") |
| A1_VERSION_STRING | Macro | Global (conditional) | Composite version string combining engine, platform, date, and version |

## Key Functions / Methods
None.

## Control Flow Notes
Compile-time header with no runtime control flow. Platform detection uses preprocessor conditionals (WIN32, __APPLE__, __MACH__, linux, __BEOS__, __NetBSD__, __OpenBSD__) to select appropriate platform macros. The A1_VERSION_STRING is only defined if not already defined elsewhere, allowing for override.

## External Dependencies
- **Preprocessor platform detection**: WIN32, __APPLE__, __MACH__, __MACOS__, linux, __BEOS__, __NetBSD__, __OpenBSD__
- **Manual sync requirement**: Comment notes that version changes must also be updated in `Resources/Aleph One Classic SDL.r`
- **Macro string concatenation**: Uses C preprocessor token pasting to compose A1_VERSION_STRING

## Notes
- Version is hardcoded to 2009-10-15; suggests this file may be outdated or archived from that release.
- Platform detection uses multiple legacy platforms (BeOS, NetBSD, OpenBSD); suggests long-term portability focus.
- Fallback platform is "Unknown" with "source" update identifier, allowing graceful degradation on unsupported platforms.
