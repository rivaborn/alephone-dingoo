# Prefix/marathon2prefix.h

## File Purpose
Prefix header for Marathon 2 providing build-time configuration, platform-specific includes, and feature flags. Included before all other source files to set compilation environment and enable/disable subsystems.

## Core Responsibilities
- Include macOS platform headers (MacHeadersCarbon.h)
- Enable/disable major subsystems (Lua, Speex, SDL_Net, QD3D/Quesa)
- Configure Mac-specific behavior (bundled app, NIBs, sheets UI)
- Set debugging and feature detection flags
- Suppress obsolete Carbon/Classic API modes

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
Compile-time only. Processed during preprocessing for all translation units. No runtime control flow.

## External Dependencies
- **MacHeadersCarbon.h** ΓÇö macOS/Carbon platform headers (Apple/Metrowerks)
- Conditional code for PowerPC vs 68K architectures (currently commented out)
- Date comment references Loren Petrich (Jan 31, 2002) suppressing `TARGET_API_MAC_CARBON` for Carbon/Classic coexistence

## Configuration Summary

| Feature | Status | Purpose |
|---------|--------|---------|
| `DEBUG` | Enabled | All debug assertions print output |
| `HAVE_QUESA` | Enabled | QD3D graphics support |
| `HAVE_LUA` | Enabled | Lua scripting engine |
| `HAVE_SDL_NET` | Enabled | SDL networking |
| `SPEEX` | Enabled | Speex audio codec |
| `USES_NIBS` | Enabled | macOS NIB resource files for UI |
| `USE_SHEETS` | Enabled | Secondary dialogs as sheet windows |
| `APPLICATION_IS_BUNDLED` | Enabled | App packaged as macOS bundle (avoids resource forks) |
| `HAVE_SNPRINTF` | Enabled | snprintf availability (CodeWarrior/GCC) |
