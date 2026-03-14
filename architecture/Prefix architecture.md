# Prefix Subsystem Overview

## Purpose
Prefix header that establishes the build-time compilation environment and configuration for Marathon 2. Included before all other source files to set platform-specific includes, enable/disable major subsystems, and manage feature detection flags.

## Key Files
| File | Role |
|------|------|
| Prefix/marathon2prefix.h | Prefix header providing build-time configuration, platform-specific includes, and feature flags |

## Core Responsibilities
- Include macOS/Carbon platform headers (MacHeadersCarbon.h)
- Enable/disable major subsystems (Lua, Speex, SDL_Net, QD3D/Quesa)
- Configure Mac-specific behavior (bundled app, NIBs, sheets UI)
- Set debugging and feature detection flags
- Suppress obsolete Carbon/Classic API modes
- Manage conditional compilation for processor architectures (PowerPC vs 68K)

## Key Interfaces & Data Flow
- **Exposes**: Preprocessor definitions, feature flags, and platform-specific includes that affect compilation of all downstream source files
- **Consumes**: Platform headers (MacHeadersCarbon.h) and architecture-specific configuration

## Runtime Role
Compile-time header only; no runtime participation in initialization, frame loop, or shutdown. Effects are confined to build environment setup.

## Notable Implementation Details
- Contains conditional code for PowerPC and 68K architectures (currently commented out)
- References Loren Petrich (Jan 31, 2002) suppression of `TARGET_API_MAC_CARBON` to manage Carbon/Classic API coexistence
