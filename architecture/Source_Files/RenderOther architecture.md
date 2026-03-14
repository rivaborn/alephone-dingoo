# Subsystem Overview

## Purpose

Source_Files/RenderOther manages all visual output for the Aleph One game engine on constrained hardware, handling screen initialization and mode switching, 2D UI rendering (HUD, menus, terminal displays), image/font resource loading, and platform-specific backends (SDL software, OpenGL, Quickdraw). This subsystem bridges high-level rendering requests from the game world layer to low-level graphics APIs while maintaining per-frame rendering orchestration.

## Key Files

| File | Role |
|------|------|
| screen.cpp/h | Display mode initialization, surface allocation, per-frame rendering pipeline orchestration, color table management |
| screen_drawing.cpp/h | Low-level 2D drawing primitives (rectangles, lines, text, shapes), clipping, port/surface management, SDL rendering backend |
| game_window.cpp/h | HUD initialization and frame update coordination, dirty-flag tracking, inventory/weapon state management, XML configuration for interface layout |
| HUDRenderer.cpp/h | Abstract HUD rendering base class, weapon/ammo/shield/oxygen display logic, motion sensor management, inventory rendering |
| HUDRenderer_SW.cpp/h | Software (SDL) HUD implementation with shape/text/rectangle rasterization |
| HUDRenderer_OGL.cpp/h | OpenGL HUD implementation with hardware-accelerated texture rendering |
| HUDRenderer_Lua.cpp/h | Lua-scripted HUD renderer with floating-point drawing API for theme customization |
| images.cpp/h | PICT/CLUT resource loading, RLE decompression, MacOS-to-SDL surface conversion, WAD file extraction |
| fades.cpp/h | Screen color effects (tint, burn, dodge, negate, randomize), gamma correction, fade state transitions, color table manipulation |
| ChaseCam.cpp/h | Third-person camera positioning with spring-damper physics, wall collision detection, camera offset switching |
| FontHandler.cpp/h | Font specification management, text width calculation, OpenGL glyph texture generation, platform-specific font loading (MacOS, SDL, TTF) |
| Shape_Blitter.cpp/h | Shape descriptor rendering with scaling, rotation, tinting, dual SDL/OpenGL backends |
| Image_Blitter.cpp/h | Image loading and rendering with rescaling, cropping, tinting support, SDL surface management |
| overhead_map.cpp/h | Minimap/automap rendering coordination, XML configuration, dispatcher to platform-specific renderers |
| OverheadMapRenderer.cpp/h | Base automap implementation with polygon/line/entity rendering, coordinate transformation, pathfinding visualization |
| OverheadMap_SDL.cpp/h | SDL software renderer for automap primitives |
| OverheadMap_OGL.cpp/h | OpenGL renderer for automap with vertex caching and batched rendering |
| computer_interface.cpp/h | Terminal display rendering, text layout, input handling, terminal state machine management, Lua callbacks |
| motion_sensor.cpp/h | Motion sensor (radar) entity tracking, position history, rendering dispatch, network compass overlays |
| ViewControl.cpp/h | Field-of-view settings (normal/extravision/tunnel-vision), teleport effects, landscape texture options, XML configuration |
| sdl_fonts.cpp/h | Font resource loading (bitmap and TrueType), caching, text measurement, styled text parsing |
| screen_shared.h | Global color tables, screen messaging, script-driven HUD element storage, on-screen debug overlay |
| OGL_Blitter.cpp/h | OpenGL 2D image blitter with surface-to-texture conversion, power-of-two tiling, scaling/rotation/tinting |
| OGL_LoadScreen.cpp/h | OpenGL-based loading screen with image display and progress bar overlay |
| TextLayoutHelper.cpp/h | Rectangle layout utility for non-overlapping UI element placement |
| TextStrings.cpp/h | Text string collection system replacing MacOS STR# resources, XML-loadable |
| FaderClassic.cpp | macOS Classic gamma table animation (platform-specific legacy code) |
| screen_definitions.h | Enum mapping screen types to resource IDs by bit depth |

## Core Responsibilities

- Initialize display modes (resolution, bit depth, windowed/fullscreen), allocate rendering surfaces, and manage OpenGL context
- Orchestrate per-frame rendering: clear buffers ΓåÆ set viewport ΓåÆ invoke world renderer ΓåÆ draw HUD ΓåÆ draw overlays ΓåÆ update screen
- Implement abstract HUD rendering with concrete backends (software/OpenGL/Lua), updating weapon, ammo, shield, oxygen, inventory, and motion sensor displays
- Load and manage 2D image resources (PICT, WAD-embedded) with format conversion, rescaling, and surface caching
- Provide text rendering with font lifecycle management (bitmap fonts, TrueType via SDL_ttf, OpenGL texture-based glyphs)
- Implement screen effects (color fades, tinting, gamma correction, tunnel vision, teleport visual feedback)
- Render minimap/automap with configurable entity markers, line visibility, polygon shading, and platform-specific backends
- Manage interactive terminal displays with rich text formatting, input handling, and group state transitions
- Display motion sensor radar with entity tracking, team compass indicators, and fade-out effects
- Provide 2D drawing primitives (shapes, images, rectangles, lines, text) for both SDL software and OpenGL hardware paths
- Control camera positioning, field-of-view transitions, and view parameters via XML configuration

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- `screen.h` ΓÇô Screen mode queries, HUD layout rectangles, color table access, rendering context, fade/effect control
- `screen_drawing.h` ΓÇô 2D drawing primitives (draw_text, draw_shape, fill_rect, frame_rect, text_width)
- `game_window.h` ΓÇô Mark HUD elements dirty, scroll inventory, update microphone state, HUD/terminal rendering requests
- `images.h` ΓÇô Load pictures/sounds from resources, rescale surfaces, platform-optimized downscaling
- `fades.h` ΓÇô Start/stop fade effects, query completion, apply environmental color tints
- `ChaseCam.h` ΓÇô Enable/disable camera, reset position, update per-frame, query final position and polygon
- `motion_sensor.h` ΓÇô Reset sensor, add entities, scan world, query entity count/state
- `overhead_map.h` ΓÇô Render automap with scale and mode parameters
- `ViewControl.h` ΓÇô Query/set FOV, teleport effects, landscape options, on-screen font
- `computer_interface.h` ΓÇô Enter terminal mode, render/handle input, serialize terminal state

**Consumes from other subsystems:**
- `world.h`, `map.h`, `player.h`, `monsters.h` ΓÇô Game state (entities, geometry, polygons, lines, endpoints)
- `render.h` ΓÇô View data, rendering context, coordinate transformations
- `network_games.h` ΓÇô Multiplayer state, player rankings, microphone status
- `shape_descriptors.h`, `interface.h` ΓÇô Shape resource IDs and bitmap definitions
- `XML_ElementParser.h` ΓÇô Configuration parsing framework
- `ColorParser.h`, `FontHandler.h` ΓÇô Color and font XML parsing
- SDL, OpenGL (conditional) ΓÇô Graphics API backend
- Resource files (WAD, image files, scenario data) ΓÇô Images, fonts, PICT resources

## Runtime Role

**Initialization phase:**
- `initialize_screen()` ΓÇô Allocate SDL surfaces (`world_pixels`, `HUD_Buffer`, `Term_Buffer`), select display mode
- `initialize_images_manager()` ΓÇô Prepare image resource subsystem
- Font initialization ΓÇô Load bitmap/TrueType fonts from resources
- HUD initialization ΓÇô Allocate HUD renderer (SW/OGL/Lua based on mode), load motion sensor graphics

**Per-frame (main loop):**
- `render_view()` ΓÇô Invoked by rendering layer to draw 3D world
- `render_screen()` ΓÇô Clear buffers, render world view to viewport, invoke HUD update and draw, overlay effects
- HUD dirty-flag updates ΓÇô `game_window_needs_update()` checks if ammo/shield/weapon/inventory changed, triggers `draw_panels()`
- Terminal mode ΓÇô If player in terminal, `handle_terminal_input()` and `draw_computer_interface()` override normal HUD
- Overhead map rendering ΓÇô Dispatch to SDL/OGL/Quickdraw renderer as configured
- Screen blit ΓÇô Copy rendered surfaces to video memory, update regions, swap OpenGL buffers

**Shutdown phase:**
- Deallocate HUD surfaces, fonts, image resources
- Cleanup OpenGL textures and display lists
- Close file handles for WAD/resource files

## Notable Implementation Details

- **Multiple HUD rendering backends** ΓÇô Abstract base class (`HUDRenderer`) with SDL software (`HUDRenderer_SW`), OpenGL (`HUDRenderer_OGL`), and Lua-scripted (`HUDRenderer_Lua`) implementations selected at runtime based on rendering mode
- **Dual 2D rendering paths** ΓÇô SDL surface blitting for software, OpenGL textured quads for hardware; `Image_Blitter` and `Shape_Blitter` abstract this difference
- **Dingoo platform optimizations** ΓÇô Fixed 320├ù240 resolution hardcoded in `screen_shared.h`, software-only rendering path (no OpenGL), optimized surface operations
- **Port-based drawing model** ΓÇô Legacy Quickdraw-era abstraction (set/restore "port" before drawing) maintained for compatibility across macOS/SDL backends
- **XML-driven configuration** ΓÇô Interface rectangles, colors, fonts, and visual parameters loaded from XML via parser callbacks rather than hardcoded
- **Platform-conditional implementations** ΓÇô Overhead map has separate SDL, OpenGL, and macOS Quickdraw renderers; conditionally compiled via `#ifdef HAVE_OPENGL`, `#ifdef mac`, etc.
- **Dirty-flag optimization** ΓÇô HUD updates tracked per element (weapon, ammo, shields) to avoid full redraws each frame; `ForceUpdate` flag forces redraw when mode changes
- **Resource caching** ΓÇô Fonts and images cached with reference counting to avoid repeated disk I/O; OpenGL textures lazy-loaded on first use
- **Coordinate transformations** ΓÇô World coordinates (fixed-point) ΓåÆ screen space (pixels) handled in rendering pipeline; 2D drawing uses abstract port coordinates for scaling/translating
