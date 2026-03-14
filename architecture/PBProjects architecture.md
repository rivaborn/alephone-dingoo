# Subsystem Overview

## Purpose

PBProjects provides build-time configuration and macOS platform initialization for the AlephOne game engine port. It declares compile-time feature availability (SDL extensions, OpenGL support) and establishes the primary entry point for macOS Cocoa integration.

## Key Files

| File | Role |
|------|------|
| config.h | Autoconf-generated feature availability flags (HAVE_OPENGL, HAVE_SDL_IMAGE, HAVE_SDL_NET); declares optional library presence and POSIX headers |
| confpaths.h | Resource and package directory path definitions (currently unused/commented) |
| precompiled_headers.h | Optimized header aggregation for macOS builds; conditionally includes SDL, Carbon, and core game engine headers |
| SDLMain.h | macOS Cocoa application controller interface; declares menu action handlers and event dispatcher |

## Core Responsibilities

- Define compile-time feature availability to gate optional subsystems throughout the codebase
- Aggregate precompiled headers to reduce macOS build times
- Declare resource path configuration points for platform-specific file access
- Provide macOS Cocoa event handling interface (menu actions, game lifecycle events)
- Bridge between Cocoa UI framework and SDL-based game engine on macOS

## Key Interfaces & Data Flow

**Exposes:**
- Feature availability flags (`HAVE_OPENGL`, `HAVE_SDL_IMAGE`, `HAVE_SDL_NET`) consumed by conditional compilation throughout Source_Files
- `SDLMain` class as entry point for macOS Cocoa event dispatch (preferences, game creation/loading/saving)
- Resource path definitions for save directory and package resource location

**Consumes:**
- Autoconf build system (generates config.h)
- SDL libraries (headers included in precompiled_headers.h)
- macOS Carbon/Cocoa frameworks (SDLMain.h, precompiled_headers.h)
- Core game engine headers (map.h, render.h, shell.h, preferences.h, network.h, sound.h, scripting.h)

## Runtime Role

On macOS: SDLMain acts as the primary application controller during initialization, dispatching Cocoa menu events to the game engine. On non-macOS platforms (including Dingoo): this subsystem provides feature availability flags only; SDLMain is not instantiated.

## Notable Implementation Details

- config.h assumes POSIX-compatible system (unistd.h); Dingoo builds use this flag set
- precompiled_headers.h targets GCC 3.0+ and is conditional on macOS (`#if defined(__APPLE__)`), not used in dingux-build cross-compile path
- confpaths.h structure exists but is not populated; paths appear to be handled elsewhere or at runtime
