# Prefix/marathon2prefix.h
## File Purpose
Prefix header for Marathon 2 providing build-time configuration, platform-specific includes, and feature flags. Included before all other source files to set compilation environment and enable/disable subsystems.

## Core Responsibilities
- Include macOS platform headers (MacHeadersCarbon.h)
- Enable/disable major subsystems (Lua, Speex, SDL_Net, QD3D/Quesa)
- Configure Mac-specific behavior (bundled app, NIBs, sheets UI)
- Set debugging and feature detection flags
- Suppress obsolete Carbon/Classic API modes

## External Dependencies
- **MacHeadersCarbon.h** ΓÇö macOS/Carbon platform headers (Apple/Metrowerks)
- Conditional code for PowerPC vs 68K architectures (currently commented out)
- Date comment references Loren Petrich (Jan 31, 2002) suppressing `TARGET_API_MAC_CARBON` for Carbon/Classic coexistence



