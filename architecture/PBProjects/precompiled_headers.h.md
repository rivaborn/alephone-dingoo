# PBProjects/precompiled_headers.h

## File Purpose
Precompiled header for AlephOne-OSX (a Marathon-like game engine) to speed up build times. Conditionally includes SDL libraries, macOS Carbon framework, and core game engine headers based on compiler version and platform flags.

## Core Responsibilities
- Conditionally includes SDL multimedia libraries (graphics, networking, image handling)
- Sets macOS-specific API targets (Carbon) for platform abstraction
- Includes all critical game engine headers (rendering, map, sound, networking, scripting, models)
- Acts as a compile-time optimization mechanism for GCC 3.0+

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `TARGET_API_MAC_CARBON` | Preprocessor macro | static | Set to 1; defines macOS Carbon API target |
| `TARGET_API_MAC_OS8` | Preprocessor macro | static | Undefined; legacy macOS 8 support disabled |
| `TARGET_API_MAC_OSX` | Preprocessor macro | static | Undefined; legacy macOS X support disabled |

## Key Functions / Methods
None.

## Control Flow Notes
Not part of runtime flow. This is a build-time mechanism. Included at compilation start to pre-process commonly used headers, reducing redundant parsing during incremental builds. The conditional compilation guards (`#ifdef __GNUG__`, `#ifdef SDL`, `#ifdef mac`) allow selective inclusion based on build configuration.

## External Dependencies
- **SDL Libraries**: SDL.h, SDL_net.h, SDL_image.h (cross-platform graphics/audio/networking)
- **macOS Framework**: Carbon/Carbon.h (legacy macOS system APIs)
- **Game Engine Headers** (defined elsewhere): cstypes.h, map.h, shell.h, render.h, OGL_Render.h, FileHandler.h, wad.h, preferences.h, scripting.h, network.h, mysound.h, csmacros.h, Model3D.h, dynamic_limits.h, effects.h, monsters.h, world.h, player.h
