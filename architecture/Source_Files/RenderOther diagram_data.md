# Source_Files/RenderOther/ChaseCam.cpp
## File Purpose
Implements third-person chase camera functionality that follows the player. The camera supports configurable offset, spring physics for smooth motion, wall collision handling, and network restrictions. Designed to provide Halo-like behind-the-player viewing.

## Core Responsibilities
- Manage chase camera activation lifecycle (enable/disable, check prerequisites)
- Maintain camera position state with history for physics calculations
- Calculate camera position based on player location with configurable offsets (behind, upward, rightward)
- Apply spring-damper physics for smooth camera movement
- Handle wall collisions using polygon/line tracing
- Provide camera state reset for level transitions and teleports
- Support horizontal camera offset switching (left/right sides)

## External Dependencies
- **map.h**: `polygon_data`, `line_data`, `endpoint_data`, geometry query functions (`get_polygon_data()`, `find_line_crossed_leaving_polygon()`, `find_adjacent_polygon()`, `find_floor_or_ceiling_intersection()`, `find_line_intersection()`, `get_line_data()`, `get_endpoint_data()`)
- **player.h**: `current_player` (global player data); `world_location3d` field access
- **network.h**: `NetAllowBehindview()` (cosmetic restriction in network games)
- **world.h**: `world_point3d`, `world_point2d`, `angle`, `world_distance` types; `translate_point3d()`, `translate_point2d()` (vector transformation)
- **ChaseCam.h**: Configuration struct `ChaseCamData`, function declarations
- **cseries.h**: Base types and macros (`TEST_FLAG`)

# Source_Files/RenderOther/ChaseCam.h
## File Purpose
Defines the interface for a third-person chase camera system in the Aleph One game engine (Marathon). Provides configuration, state management, and per-frame update logic for a Halo-like camera that follows the player with configurable physics-based positioning and damping.

## Core Responsibilities
- Define chase camera configuration data structure with offset and physics parameters
- Provide state query and control functions (active/inactive toggles)
- Manage camera initialization and reset on level transitions
- Update camera position each frame with spring/damping physics
- Calculate final camera position, orientation (yaw/pitch), and world polygon for rendering
- Support optional camera-switching based on horizontal offset

## External Dependencies
- `world.h` ΓÇö provides `world_point3d`, `angle` typedef, and world coordinate system definitions

# Source_Files/RenderOther/computer_interface.cpp
## File Purpose
Manages in-game computer terminal interfaces, including rendering, input handling, and state tracking. Handles terminal text display with rich formatting (colors, styles), group transitions (logon/information/teleport), and serialization of terminal data loaded from maps.

## Core Responsibilities
- Initialize and maintain per-player terminal state machines
- Load and unpack terminal data from maps (groupings, text, style changes)
- Render terminal screens with borders, text, images (pictures/videos), and checkpoints
- Parse player input (arrow keys, page up/down, space, escape) during terminal mode
- Calculate text layout (line breaks, word wrapping) based on font metrics
- Serialize/deserialize terminal state for save/load operations
- Handle terminal group transitions (logon ΓåÆ information ΓåÆ teleport sequences)
- Support Lua callbacks for terminal entry/exit events
- Manage text styling (bold, italic, underline, color) via embedded markup

## External Dependencies
- **Notable includes:** `world.h`, `map.h`, `player.h` (game state), `screen_drawing.h` (rendering), `SoundManager.h`, `lua_script.h` (Lua integration), `sdl_fonts.h` (font metrics), `Packing.h` (byte stream utilities), `interface.h` (error strings)
- **External symbols used:** 
  - `dynamic_world`, `static_world` (game state)
  - `get_player_data()`, `get_indexed_terminal_data()` (defined elsewhere)
  - `play_object_sound()`, `Sound_TerminalLogon()` (audio)
  - `calculate_destination_frame()`, `_get_font_line_height()`, `_get_interface_color()` (rendering)
  - `L_Call_Terminal_Exit()` (Lua)
  - `world_pixels`, `draw_surface` (SDL surfaces)

# Source_Files/RenderOther/computer_interface.h
## File Purpose
Defines the interface for the in-game computer terminal system, which renders and manages interactive terminal displays for players (briefing text, status messages, menus). Handles player input during terminal interaction, terminal state serialization, and text preprocessing/formatting.

## Core Responsibilities
- Initialize and manage terminal mode per player
- Render terminal display with formatted text, embedded formatting codes, and media references
- Process player input and actions while in terminal mode  
- Serialize/deserialize terminal state for persistence across map/level loads
- Preprocess and encode terminal text resources (at build time)
- Manage terminal data packing (player state vs. map-embedded data)

## External Dependencies
- Standard fixed-width integer types (`int16`, `uint32`, `uint8`, `bool`, `byte`) ΓÇö defined elsewhere
- `#ifdef PREPROCESSING_CODE` gates build-time text processing functions (used by offline tool)
- GPL license header indicates Aleph One / Marathon engine origin

# Source_Files/RenderOther/FaderClassic.cpp
## File Purpose
Implements direct video device gamma table manipulation to animate screen color lookup tables on macOS Classic. Part of a color fading system that works in Carbon environments by bypassing high-level APIs and directly accessing the graphics device driver.

## Core Responsibilities
- Exports `animate_screen_clut_classic()` function to apply new color tables via gamma manipulation
- Retrieves current gamma table from video device driver using Control status calls
- Converts 16-bit packed RGB color data to planar format (all reds, then greens, then blues)
- Handles both high bit-depth (>8 bits) and low bit-depth (Γëñ8 bits) gamma table entries
- Validates device state (Control driver loaded, color capable, valid gamma table)

## External Dependencies
- **macOS Carbon/Classic APIs:** Control, PBStatus, GDHandle device structures, Devices.h, Quickdraw.h, Video.h
- **Utility macros:** MIN, MAX (from csmacros.h)
- **Platform:** macOS Classic only; requires working Control device driver and QuickDraw graphics subsystem

# Source_Files/RenderOther/fades.cpp
## File Purpose
Implements the screen fading and color-tinting system for the Aleph One game engine. Manages visual effects like cinematic fades, damage flashes, and environmental tints (water, lava, sewage) using both CPU color-table manipulation and GPU-accelerated OpenGL rendering.

## Core Responsibilities
- Manage fade state transitions with time-based interpolation
- Apply six distinct color blending algorithms (tint, burn, dodge, negate, randomize, soft-tint)
- Support environmental effect layering (e.g., tint blue while underwater)
- Enforce fade priority system to prevent conflicting effects
- Provide both software (color-table) and hardware (OpenGL) rendering paths
- Parse and apply XML-based fade definitions at runtime
- Implement gamma correction for display calibration

## External Dependencies
- **cseries.h**: Macros (`CEILING`, `FLOOR`, `PIN`, `MAX`), memory, `GetMemberWithBounds()`, type definitions
- **fades.h**: Public API, fade/effect type enums, constants
- **screen.h**: Global `world_color_table`, `visible_color_table`; `animate_screen_clut()` function
- **ColorParser.h**: `Color_GetParser()`, `Color_SetArray()` for XML color parsing
- **OGL_Faders.h** [ifdef HAVE_OPENGL]: `OGL_Fader`, `OGL_FaderActive()`, `GetOGL_FaderQueueEntry()`
- **Music.h**: `Music::instance()->Idle()` called during `full_fade()` to keep audio responsive
- Standard library: `<string.h>`, `<stdlib.h>`, `<math.h>` (pow), `<limits.h>` (SHRT_MAX)

**Defined elsewhere**:
- `machine_tick_count()`: Get engine tick counter
- `obj_copy()`: Macro for struct copy
- `XML_ElementParser`, `ReadInt16Value()`, etc.: XML parsing framework
- `assert()`, various validation helpers from cseries

# Source_Files/RenderOther/fades.h
## File Purpose
Header file defining the fade and screen effect system for the Aleph One game engine. Manages cinematic fades, visual feedback effects (damage flashes, item pickups, teleports), and environmental color tints, with support for gamma correction and XML-based configuration.

## Core Responsibilities
- Define fade type enumeration (cinematic, damage, environmental tints, etc.)
- Declare fade lifecycle functions (initialize, update, start, stop, check completion)
- Provide fade effect configuration and timing queries
- Support color table manipulation and gamma correction
- Expose XML parser interface for fade configuration
- Workaround for MacOS-specific screen painting bugs via effect delay

## External Dependencies
- **`"XML_ElementParser.h"`** ΓÇö XML parsing framework for configuration; provides `XML_ElementParser` base class used by `Faders_GetParser()`
- **`color_table` struct** ΓÇö Defined elsewhere; used for gamma correction and color palette management
- Standard C library (stdio implicit in included headers)

# Source_Files/RenderOther/FontHandler.cpp
## File Purpose
Implements font specification, management, and rendering for the Aleph One game engine. Supports platform-specific font handling (MacOS/SDL) and OpenGL-accelerated text rendering. Manages font parameters, metrics, textures, and provides XML configuration parsing.

## Core Responsibilities
- Font parameter management (name, size, style, file path) with platform-specific initialization
- Text width calculation and character glyph width lookup tables
- OpenGL font texture generation with glyph packing and display list creation
- Platform-specific text rendering (MacOS Quickdraw, SDL surfaces, OpenGL)
- Font name list parsing with fallback/preference support
- Centralized cleanup via font registry for all active OpenGL fonts
- XML configuration parsing for batch font attribute updates

## External Dependencies
- **OpenGL:** glGenTextures, glBindTexture, glTexImage2D, glGenLists, glNewList, glCallList, glPushMatrix, glPopMatrix, glTranslatef/d, glBegin/glEnd, glTexCoord2f, glVertex2d, glPushAttrib, glPopAttrib, glEnable/glDisable
- **MacOS:** TextSpec, TextFont, TextFace, TextSize, GetFontInfo, GetFNum, NewGWorld, GetGWorldPixMap, LockPixels, GetGWorld, SetGWorld, BackColor, ForeColor, EraseRect, MoveTo, DrawChar, DisposeGWorld, Rect, GWorldPtr, PixMapHandle, GetPixBaseAddr
- **SDL:** SDL_Surface, SDL_CreateRGBSurface, SDL_FillRect, SDL_MapRGB, SDL_FreeSurface; load_font, unload_font, draw_text, char_width (from sdl_fonts.h / screen_drawing.cpp)
- **Game engine:** screen_rectangle (screen_drawing.h), XML_ElementParser (base class), shape_descriptors.h (included but unused here)

# Source_Files/RenderOther/FontHandler.h
## File Purpose
Defines the FontSpecifier class for managing font specifications, metrics, and rendering across MacOS/SDL/OpenGL platforms. Handles font parameter specification, metric derivation, and multi-platform text rendering including OpenGL texture-based font rendering.

## Core Responsibilities
- Store and manage font parameters (name fallback list, size, style, line height adjustment)
- Compute and cache derived font metrics (height, line spacing, ascent/descent/leading, per-character widths)
- Provide text width queries for layout operations (centering, truncation)
- Render text in OpenGL using pre-built font textures and display lists
- Support platform-specific font systems (MacOS QuickDraw, SDL, OpenGL)
- Parse and update font specifications from XML configuration
- Manage global font registry for OpenGL resource cleanup

## External Dependencies
- **cseries.h:** Core types, platform macros (mac, SDL, HAVE_OPENGL), Str255, struct Rect
- **XML_ElementParser.h:** Font_GetParser(), Font_SetArray() for XML-driven font configuration
- **sdl_fonts.h:** font_info class (when SDL defined)
- **GL/gl.h:** OpenGL types/functions (GLuint, glPushMatrix, etc.) when HAVE_OPENGL defined
- **\<set\>:** STL container for m_font_registry

# Source_Files/RenderOther/game_window.cpp
## File Purpose
Manages the game window HUD (Heads-Up Display) for both software and OpenGL rendering modes. Handles HUD initialization, frame updates, weapon/ammo display state tracking, inventory navigation, and XML-driven configuration of interface element layouts (weapons, ammo counters, colors, fonts).

## Core Responsibilities
- Initialize HUD buffer for software rendering (SDL surface)
- Update and draw interface panels each frame, respecting dirty-flag optimization
- Track and propagate HUD element dirty states (weapon, ammo, shields, oxygen)
- Manage player inventory screen navigation and scrolling
- Parse and apply XML configuration for weapon/ammo display positioning and appearance
- Initialize motion sensor display with graphics resources
- Coordinate rendering between software (SDL) and OpenGL paths
- Manage microphone state for network audio

## External Dependencies
- `cseries.h` ΓÇö core engine types, macros
- `HUDRenderer_SW.h` ΓÇö `HUD_SW_Class` (software HUD renderer)
- `game_window.h` ΓÇö public interface (declarations)
- `ColorParser.h`, `FontHandler.h` ΓÇö XML-based config
- `screen.h` ΓÇö `alephone::Screen` singleton, rendering mode queries
- `shell.h` ΓÇö shell utilities, screen_mode_data
- `preferences.h` ΓÇö game settings
- `images.h` ΓÇö `get_picture_resource_from_images()`, `picture_to_surface()`
- `network_sound.h` ΓÇö `set_network_microphone_state()`
- SDL headers (via cseries.h) ΓÇö `SDL_Surface`, `SDL_Rect`, `SDL_BlitSurface()`, etc.
- OpenGL headers (conditional) ΓÇö `gl.h` / `GL/gl.h`

**Defined elsewhere** (called but not defined here):
- `initialize_motion_sensor()`, `reset_motion_sensor()`
- `draw_panels()` (forward-declared, then re-defined later in same file)
- `validate_world_window()`
- `game_window_is_full_screen()`
- `get_player_data()`, `calculate_player_item_array()`, `get_item_kind()`
- `SET_INVENTORY_DIRTY_STATE()`, `GET_CURRENT_INVENTORY_SCREEN()`, `GET_GAME_OPTIONS()`
- `RequestDrawingHUD()`, `RequestDrawingTerm()`
- `mark_collection_for_loading/unloading()`
- `alert_user()`, `_set_port_to_HUD()`, `_restore_port()`
- Color/font parsing utilities

# Source_Files/RenderOther/game_window.h
## File Purpose
Public interface for game window initialization and HUD/interface rendering in the Aleph One game engine. Declares functions for drawing, updating, and managing the state of on-screen UI elements including ammo/shield/oxygen displays, inventory, and network stats.

## Core Responsibilities
- Initialize and manage the game rendering window
- Draw and update the HUD each frame
- Mark HUD elements as "dirty" to trigger redraw (ammo, shield, oxygen, weapons, inventory)
- Scroll player inventory
- Manage microphone recording state
- Provide access to XML parser for interface configuration

## External Dependencies
- `<Rect>` ΓÇô graphics/geometry primitive (defined elsewhere)
- `XML_ElementParser` ΓÇô configuration/parsing subsystem (defined elsewhere)
- OpenGL (inferred from `OGL_DrawHUD` naming convention)

# Source_Files/RenderOther/HUDRenderer.cpp
## File Purpose
Core implementation of HUD rendering for the Marathon-like game engine (Aleph One). Manages all on-screen UI elements including energy/oxygen bars, weapon and ammunition displays, inventory panels, and network player information. Supports both standard HUD and a Lua-driven texture palette viewer for debugging/asset previewing.

## Core Responsibilities
- Update all HUD elements conditionally based on dirty flags (energy, oxygen, weapon, ammo, inventory)
- Render progress bars (shield/energy and oxygen) with multi-level states
- Display weapon panel with single/multi-weapon variations and naming
- Render ammunition counts as grid layouts or energy bars
- Manage inventory display with category headers, item lists, and validity indicators
- Draw network-mode overlays (player rankings, game timer, kill counter)
- Support optional Lua texture palette viewer (gp2x/dingoo platform hack)

## External Dependencies
- **Notable includes:** `HUDRenderer.h` (class definition), `lua_script.h` (conditional Lua texture palette), `config.h` (HAVE_LUA feature flag)
- **Implied externs:** `current_player`, `current_player_index`, `dynamic_world`, `interface_state`, `weapon_interface_definitions`, game query functions (`get_player_desired_weapon`, `calculate_player_item_array`, `get_player_weapon_ammo_count`, etc.)
- **Virtual methods (derived class):** `DrawShape`, `DrawShapeAtXY`, `DrawText`, `FillRect`, `FrameRect`, `DrawTexture`, `SetClipPlane`, `DisableClipPlane`, `update_motion_sensor`, `render_motion_sensor`, `draw_all_entity_blips`

# Source_Files/RenderOther/HUDRenderer.h
## File Purpose
Base class and interface for in-game HUD rendering. Defines the abstract contract that renderer implementations (OpenGL, software, etc.) must fulfill to display the suit interface, weapon panels, motion sensor, and overlay elements during gameplay.

## Core Responsibilities
- Coordinate frame-by-frame HUD state updates (energy, oxygen, weapons, inventory)
- Manage dirty-flag tracking to avoid redundant redraws of unchanged UI elements
- Render suit energy/oxygen bar graphics and motion sensor display
- Display active weapon panel with ammo counts and magazine visuals
- Render player inventory, motion sensor entity blips, and network compass
- Define abstract drawing primitives for platform-specific rendering backends
- Handle message area display and microphone state indication

## External Dependencies
- **Player state:** accesses `player.h` (suit energy, oxygen, weapons, inventory)
- **Weapons & Items:** `weapons.h`, `items.h` for item definitions and ammo counts
- **World:** `map.h`, `world.h` for motion sensor entity locations
- **Sound:** `SoundManager.h` for button click feedback
- **Drawing:** `screen_drawing.h` for screen coordinates and rectangles
- **Network:** `network_games.h` for multiplayer-specific HUD elements
- **Texture IDs:** shape descriptors (panel art, bars, motion sensor mounts/icons) defined in enums at file top

**Notes:** Texture enum values (_energy_bar, _magnum_panel, etc.) are hardcoded indices into a resource system loaded elsewhere. The class is designed as an abstract interface to support multiple rendering backends without duplicating high-level HUD logic.

# Source_Files/RenderOther/HUDRenderer_Lua.cpp
## File Purpose
Implements Lua-scripted HUD rendering for Aleph One. Provides a C++ bridge layer that exposes drawing primitives (rectangles, text, images, shapes) to Lua scripts, with dual OpenGL and SDL backend support.

## Core Responsibilities
- **Lifecycle management**: Initialize and finalize drawing state for each frame (`start_draw()`, `end_draw()`)
- **Motion sensor**: Update and manage entity detection blips from the motion sensor
- **Clip regions**: Apply scissor/clip rectangles for constrained drawing areas
- **Primitive rendering**: Draw filled/outlined rectangles, text, images, and shapes in both OpenGL and SDL
- **Backend abstraction**: Provide unified drawing API with conditional OpenGL or SDL implementation
- **Entity blips**: Maintain a list of motion-sensor blips (radar contacts) that Lua can query

## External Dependencies
- **FontHandler.h**: `FontSpecifier` (font metrics, rendering)
- **Image_Blitter.h**: `Image_Blitter` (image drawing)
- **Shape_Blitter.h**: `Shape_Blitter` (shape/bitmap drawing)
- **lua_hud_script.h**: `L_Call_HUDDraw()` (Lua script entry point)
- **shell.h**: `GET_GAME_OPTIONS()`, `MotionSensorActive` (game state)
- **screen.h**: `alephone::Screen::instance()` (window/clip rects)
- **OGL_Setup.h**: `Using_sRGB` flag, OpenGL color setup
- **SDL**: `SDL_Surface`, `SDL_Rect`, `SDL_FillRect()`, `SDL_BlitSurface()`, video surface access
- **OpenGL** (conditional): `glPushAttrib()`, `glVertex2f()`, `glScissor()`, etc.
- **math.h**: `arctangent()` (used in `add_entity_blip()`)
- Defined elsewhere: `motion_sensor_scan()`, `reset_motion_sensor()`, `current_player_index`

# Source_Files/RenderOther/HUDRenderer_Lua.h
## File Purpose
Defines a Lua-integrated HUD renderer class (`HUD_Lua_Class`) that extends the base `HUD_Class` to provide Aleph One game engine HUD rendering via Lua scripts. Manages motion sensor blips, draws geometric primitives (rectangles, text, images, shapes), and mediates between Lua themes and the underlying rendering backend. Conditionally compiled when Lua support is enabled (`HAVE_LUA`).

## Core Responsibilities
- **Blip management**: Track, update, and query motion sensor entity blips with spatial/intensity data
- **Drawing surface**: Maintain OpenGL/SDL rendering surface state and clipping regions
- **Lua drawing API**: Expose filling/framing rectangles, rendering text/images/shapes at arbitrary coordinates
- **Motion sensor rendering**: Update and draw motion sensor display each frame
- **Base class overrides**: Implement pure virtual drawing methods from `HUD_Class` (mostly as empty stubs for Lua compatibility)

## External Dependencies
- **Includes**: `config.h` (feature flags), `HUDRenderer.h` (base class, HUD constants, world types)
- **Forward declarations**: `FontSpecifier`, `Image_Blitter`, `Shape_Blitter` (rendering primitives)
- **SDL types**: `SDL_Surface`, `SDL_Rect` (software rendering backend)
- **Inherited from base**: `HUD_Class` (update loop, motion sensor, inventory, weapon display)
- **Game engine types**: `shape_descriptor`, `screen_rectangle`, `world_distance`, `angle`, `point2d` (defined elsewhere in game engine)

---

**Notes:**
- Most protected virtual overrides (e.g., `DrawShape`, `FillRect`, `DrawText`) are **empty stubs** because Lua HUD uses the public floating-point API instead.
- Lua access to blips is read-only via `entity_blip_count()` and `entity_blip(index)`; Lua cannot directly modify blips after adding them.

# Source_Files/RenderOther/HUDRenderer_OGL.cpp
## File Purpose
OpenGL-based HUD renderer for the Aleph One game engine. Responsible for rendering the player-facing heads-up display showing weapons, ammo, shields, oxygen, inventory, motion sensor, and messages each frame.

## Core Responsibilities
- Load and draw the static HUD backdrop panel
- Render dynamic HUD elements (weapon/ammo/shield/oxygen displays, inventory)
- Draw shaped interface graphics with texture support and scaling
- Render text labels and messages with specified fonts and colors
- Fill and frame rectangles for UI elements
- Manage motion sensor display with circular clipping regions
- Set up OpenGL state and viewport for HUD rendering

## External Dependencies
- **OpenGL:** `GL/gl.h` or `OpenGL/gl.h` (platform-conditional)
- **FontHandler:** `FontSpecifier` for text rendering; `get_interface_font()`
- **game_window:** Functions to mark HUD elements dirty (`mark_*_display_as_dirty()`)
- **OGL infrastructure:** `OGL_Setup.h`, `OGL_Textures.h`, `OGL_Blitter`, `Shape_Blitter`, `TextureManager`
- **Images/Resources:** `get_shape_bitmap_and_shading_table()`, `INTERFACE_PANEL_BASE` constant
- **Interface palette:** `get_interface_color()`, `get_interface_rectangle()`
- **Lua:** `LuaTexturePaletteSize()` for dynamic palette override
- **Render state:** `_shading_normal`, `_shadeless_transfer`, `OGL_Txtr_WeaponsInHand`
- **Imported types:** `shape_descriptor`, `screen_rectangle`, `Rect`, `SDL_Rect`, `rgb_color`

# Source_Files/RenderOther/HUDRenderer_OGL.h
## File Purpose
OpenGL-specific HUD renderer implementation. Declares `HUD_OGL_Class`, which inherits from the base `HUD_Class` and provides concrete OpenGL rendering methods for the game's heads-up display (motion sensor, weapons panel, inventory, etc.).

## Core Responsibilities
- Override virtual rendering methods with OpenGL implementations
- Handle motion sensor updates and rendering
- Draw and manage entity blips (enemy/ally markers) on the HUD
- Provide texture and shape drawing via OpenGL
- Support dynamic clipping planes for bounded HUD elements
- Implement text and rectangle rendering (fill/frame)

## External Dependencies
- `HUDRenderer.h` ΓÇö base class (`HUD_Class`) and shared HUD constants/structures
- `config.h` ΓÇö conditional compilation (`HAVE_OPENGL` guard)

# Source_Files/RenderOther/HUDRenderer_SW.cpp
## File Purpose
Software-rendered HUD implementation for the Aleph One game engine using SDL surfaces. Provides concrete implementations of HUD rendering methods including motion sensor updates, shape and texture drawing, text rendering, and basic rasterization primitives (rectangles).

## Core Responsibilities
- Update and render the motion sensor display with state change detection
- Draw shapes from the shape collection at specified screen coordinates
- Render textured shapes with automatic aspect-ratio-preserving scaling using Shape_Blitter
- Draw screen text with font and color control
- Rasterize filled and outlined rectangles
- Provide SDL surface rotation utility for pixel data transformation

## External Dependencies
- **SDL:** `SDL_Surface`, `SDL_Rect`, `SDL_CreateRGBSurface()`, `SDL_SetColors()`
- **images.h:** Image/resource management (implied by included files)
- **shell.h:** `get_shape_surface()` (commented in include as "get_shape_surface!?")
- **Shape_Blitter.h:** `Shape_Blitter` class for textured shape rendering
- **Defined elsewhere:** `_draw_screen_shape()`, `_draw_screen_shape_at_x_y()`, `_draw_screen_text()`, `_fill_rect()`, `_frame_rect()`, `reset_motion_sensor()`, `motion_sensor_scan()`, `motion_sensor_has_changed()`, `get_interface_rectangle()`, `GET_GAME_OPTIONS()`, `GET_DESCRIPTOR_*()` macros, `BUILD_DESCRIPTOR()` macro
- **Global state:** `HUD_Buffer` (extern SDL_Surface*), `MotionSensorActive` (extern bool), `current_player_index` (implied from `reset_motion_sensor()` call), `ForceUpdate` (class member flag)

# Source_Files/RenderOther/HUDRenderer_SW.h
## File Purpose
Software-rendering implementation of the HUD renderer for the Aleph One game engine. Provides a concrete subclass that renders the heads-up display (weapon panels, ammo, motion sensor, player status) using CPU-based graphics primitives instead of hardware acceleration.

## Core Responsibilities
- Implement software-rendered HUD display pipeline
- Render the motion sensor (radar) with entity tracking
- Draw weapon panels, ammo counters, and inventory display
- Provide primitive drawing operations: shapes, text, filled/framed rectangles
- Handle entity blips (radar contacts) for motion sensor
- Manage clip plane constraints for rendering regions (Windows platform)

## External Dependencies
- **HUDRenderer.h** ΓÇö Base class definition; defines pure virtual methods and common HUD update logic
- **shape_descriptor**, **screen_rectangle**, **point2d** ΓÇö Types defined elsewhere (likely screen_drawing.h)
- **Platform macros** ΓÇö Windows-specific `#undef DrawText` (Windows SDK defines DrawText as a macro; this prevents collision with the virtual method)

**Notes:** All method implementations are deferred to the `.cpp` file. `SetClipPlane()` and `DisableClipPlane()` are stub implementations (likely clipping support is not critical for software rendering).

# Source_Files/RenderOther/Image_Blitter.cpp
## File Purpose
Implements the `Image_Blitter` class, a wrapper around SDL surfaces that loads, manages, and renders 2D images for UI purposes in the Aleph One game engine. Provides scaling, cropping, and format-conversion utilities for cross-platform 2D rendering.

## Core Responsibilities
- Load images from multiple sources: `ImageDescriptor` objects, picture resource IDs, and raw SDL surfaces
- Manage SDL surface lifecycle (allocation, caching, deallocation) to optimize memory and rendering performance
- Handle image rescaling and maintain cached scaled surfaces to avoid recomputation during repeated renders
- Support proportional cropping region adjustments when image dimensions change
- Draw images to destination surfaces with configurable source/destination rectangles
- Query image dimensions in both original and scaled states

## External Dependencies
- `#include <SDL/SDL.h>` ΓÇö Cross-platform 2D graphics library
- `#include "Image_Blitter.h"` ΓÇö Class header
- `#include "images.h"` ΓÇö Game engine image resource system
- Functions from `images.h`: `get_picture_resource_from_images()`, `picture_to_surface()`, `rescale_surface()`
- Classes from external headers: `ImageDescriptor` (ImageLoader.h), `LoadedResource` (cseries.h)

# Source_Files/RenderOther/Image_Blitter.h
## File Purpose
Declares the `Image_Blitter` class for loading and rendering 2D images/sprites in the UI layer. Handles image transformation (tinting, rotation, scaling) and blitting to SDL surfaces with flexible source/destination rectangles and cropping.

## Core Responsibilities
- Load images from multiple sources: `ImageDescriptor`, picture resource IDs, and SDL surfaces
- Manage image lifecycle (load, unload, track loaded state)
- Apply transformations: tinting (RGBA), rotation (degrees), scaling, and cropping
- Draw images to destination surfaces with source/dest rect control
- Maintain both original and scaled surface copies for rendering efficiency
- Query image dimensions (scaled and unscaled)

## External Dependencies
- **SDL/SDL.h** ΓÇô Graphics library for surfaces and rectangles
- **ImageLoader.h** ΓÇô ImageDescriptor class for image data container
- **cseries.h** ΓÇô General utilities and type definitions (standard Aleph One utility header)

**External symbols used:**
- `SDL_Surface`, `SDL_Rect` ΓÇô Defined in SDL library
- `ImageDescriptor` ΓÇô Defined in ImageLoader.h

# Source_Files/RenderOther/images.cpp
## File Purpose
Manages loading, decompressing, and displaying image resources (pictures and color tables) from the game's resource and WAD files. Handles picture format conversion between Mac PICT format and SDL surfaces, with support for RLE decompression and various color depths.

## Core Responsibilities
- Load and manage PICT/CLUT resources from both MacOS resource forks and Win32-compatible WAD files
- Decompress PackBits RLE-encoded picture data at 8, 16, and 32-bit depths
- Convert MacOS PICT resources to SDL surfaces for rendering
- Handle color table (CLUT) resource management and color model conversion
- Provide high-level API for retrieving and displaying full-screen pictures with optional scrolling
- Support resource ID mapping based on display bit depth (8/16/32-bit fallback logic)

## External Dependencies
- **FileHandler.h:** `FileSpecifier`, `OpenedFile`, `OpenedResourceFile`, `LoadedResource` (object-oriented file I/O abstraction)
- **wad.h:** WAD file structures and functions (`wad_data`, `read_indexed_wad_from_file()`, `extract_type_from_wad()`, `free_wad()`)
- **screen_drawing.h:** Clipping state (`draw_clip_rect_active`, `draw_clip_rect`); color/shading utilities
- **SDL, SDL_image:** `SDL_RWops`, `SDL_Surface`, `SDL_BlitSurface()`, `IMG_LoadTyped_RW()` (rendering and image decompression)
- **byte_swapping.h:** `byte_swap_memory()` for endian conversion
- **cseries.h, interface.h:** Macros (FOUR_CHARS_TO_INT), constants, memory management
- **screen.h:** `interface_bit_depth` (external global); called by shell for full-screen image display

# Source_Files/RenderOther/images.h
## File Purpose
Header for the images resource subsystem in Aleph One (Marathon engine port). Declares functions to load, manage, and render picture/sound/text resources from scenario and images files, supporting MacOS resource forks, SDL surfaces, and platform-specific optimizations.

## Core Responsibilities
- Initialize and manage images subsystem (`initialize_images_manager`)
- Check for picture existence in images file or scenario file
- Load picture/sound/text resources into `LoadedResource` wrapper objects
- Calculate and build color lookup tables (CLUTs) from picture data
- Render full-screen pictures from resource files to display
- Scroll animated picture sequences with optional text blocks
- Convert MacOS PICT resources to SDL_Surface for cross-platform rendering
- Provide surface utilities: rescale, tile, platform-specific downscaling (Dingoo)

## External Dependencies
- `FileHandler.h` ΓÇö provides `FileSpecifier`, `LoadedResource` classes
- Implicit: `color_table` type (defined elsewhere in codebase)
- Implicit: SDL library (conditional; `SDL_Surface`, `SDL_RWops`)
- Implicit: MacOS Carbon/Resources (on Mac builds)
- `tags.h` ΓÇö typecode constants (via FileHandler.h)


# Source_Files/RenderOther/motion_sensor.cpp
## File Purpose
Implements the motion sensor HUD displayΓÇöa circular radar that tracks nearby monsters and players. Manages entity detection, position history, rendering with fading intensity effects, and network compass indicators for team awareness in multiplayer games.

## Core Responsibilities
- **Entity lifecycle management**: Add/remove entities from tracking list with smooth fade-out on removal
- **Spatial scanning**: Periodically scan world for monsters/players within range, check line-of-sight
- **Position tracking**: Maintain 6-frame history of entity positions for distance-based intensity visualization
- **Rendering**: Draw entity blips as transparent sprites on circular sensor; handle platform-specific renderers (SW/OpenGL/Lua)
- **Network compass**: Display directional indicators for team members in multiplayer
- **XML configuration**: Allow runtime customization of sensor parameters and monsterΓåÆdisplay-type mappings

## External Dependencies
- **map.h**: `world_point3d`, `world_point2d`, `world_distance`, `WORLD_ONE`, `guess_distance2d()`, `MAXIMUM_MONSTERS_PER_MAP`, `get_object_data()`, object structures
- **monsters.h**: `monster_data`, `MAXIMUM_MONSTERS_PER_MAP`, `SLOT_IS_USED()`, `MONSTER_IS_PLAYER()`, `MONSTER_IS_ACTIVE()`, `get_monster_data()`, monster type enums
- **player.h**: `player_data`, `get_player_data()`, team constants
- **network_games.h**: `get_network_compass_state()`, compass bit flags
- **interface.h**: `get_shape_bitmap_and_shading_table()`, `shape_descriptor`, `bitmap_definition`, shape loading
- **render.h**: View/rendering infrastructure
- **HUDRenderer_*.h**: Platform-specific rendering class definitions (HUD_Class, HUD_SW_Class, HUD_OGL_Class, HUD_Lua_Class)
- Standard C: `math.h`, `string.h` (memset, memmove), `stdlib.h`

# Source_Files/RenderOther/motion_sensor.h
## File Purpose
Interface for the motion sensor HUD display in the Aleph One game engine. Defines display type enums, initialization/management functions, and XML configuration parsing for the in-game motion sensor that tracks nearby entities.

## Core Responsibilities
- Define entity type categories (friendly, alien, enemy) for motion sensor display
- Initialize motion sensor graphics with shape descriptors for each entity type
- Manage motion sensor state (reset, change detection, range adjustment)
- Provide XML parser integration for loading motion sensor configuration
- Track state changes to optimize rendering updates

## External Dependencies
- `#include "XML_ElementParser.h"` ΓÇô XML configuration parsing infrastructure
- `shape_descriptor` type (defined elsewhere) ΓÇô graphics resource references
- Entity/monster system (undefined here) ΓÇô implicit dependency via monster_index parameter

# Source_Files/RenderOther/OGL_Blitter.cpp
## File Purpose
OpenGL implementation of a 2D image blitter for rendering SDL surfaces as textured quads. Handles surface-to-texture conversion, tiling for size limits, and rendering with scaling, rotation, and color tinting. Part of the Aleph One engine's 2D rendering pipeline.

## Core Responsibilities
- Load SDL surfaces into OpenGL textures, applying power-of-two padding and edge smearing
- Tile large surfaces across multiple textures (max 256├ù256) to respect hardware limits
- Render textured quads to screen with transformations (scale, rotate, tint, crop)
- Manage texture lifecycle and lazy-load on first draw
- Track active blitters via static registry for coordinated cleanup
- Set up orthographic projection matrix for 2D rendering over 3D scene

## External Dependencies
- **SDL:** `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`, `SDL_SetAlpha()`, `SDL_GetVideoSurface()`, `SDL_Rect`, `SDL_Surface`
- **OpenGL:** `glGenTextures()`, `glBindTexture()`, `glTexParameteri()`, `glTexImage2D()`, `glDeleteTextures()`, matrix/viewport/blend calls
- **Engine:** `Image_Blitter` (parent class, defined elsewhere), `OGL_Setup.h` (config access via `Get_OGL_ConfigureData()`, sRGB global `Using_sRGB`), `OGL_Textures.cpp` (`NextPowerOfTwo()`)
- **Standard:** `<vector>`, `<set>`, `<cmath>` (implicit via includes)

# Source_Files/RenderOther/OGL_Blitter.h
## File Purpose
Defines `OGL_Blitter`, an OpenGL-based image blitter for rendering SDL surfaces and ImageDescriptor objects to screen. Inherits from `Image_Blitter` and manages OpenGL texture tiling, loading, and rendering operations.

## Core Responsibilities
- Manage OpenGL texture creation, binding, and deletion
- Tile large images into fixed-size (256├ù256) textures for GPU memory efficiency
- Provide overloaded `Draw()` methods for rendering to destination rectangles
- Maintain a global registry of all active `OGL_Blitter` instances
- Expose static utility methods for screen resolution and global texture management
- Handle platform-specific OpenGL header inclusion (macOS/Linux/Windows)

## External Dependencies
- **Headers:** `cseries.h` (core types), `ImageLoader.h` (ImageDescriptor), `Image_Blitter.h` (base class)
- **SDL:** `SDL_Surface`, `SDL_Rect`, `SDL.h`
- **OpenGL:** `gl.h`, `glu.h`, `glext.h` (platform-conditional includes; HAVE_OPENGL guard)
- **STL:** `<vector>` (m_rects, m_refs), `<set>` (m_blitter_registry)
- **Symbols from elsewhere:** `Image_Blitter` base class (Load, crop_rect, m_surface inheritance)

# Source_Files/RenderOther/OGL_LoadScreen.cpp
## File Purpose
Implements an OpenGL-based loading screen that displays images with optional progress bars during engine/game loading. Part of the Aleph One game engine's rendering system, managing screen display and cleanup during initialization phases.

## Core Responsibilities
- Singleton pattern management for global loading screen access
- Image loading from files and GPU texture preparation
- Screen-space rendering with aspect-ratio-aware scaling/stretching
- Progress bar overlay with color customization
- OpenGL state management (matrix transforms, texture binding, vertex drawing)
- Resource cleanup and screen buffer swaps

## External Dependencies
- **Includes (this file):**
  - `OGL_LoadScreen.h` ΓÇö class definition
  - `screen.h` ΓÇö screen surface API
  - `OGL_Setup.h` ΓÇö `SglColor3us()` macro/function

- **External symbols used:**
  - `OGL_SwapBuffers()`, `OGL_ClearScreen()`, `bound_screen()` ΓÇö defined elsewhere, render API
  - `SDL_GetVideoSurface()` ΓÇö SDL 1.x video API
  - OpenGL: `glMatrixMode()`, `glPushMatrix()`, `glTranslated()`, `glScaled()`, `glDisable()`, `glEnable()`, `glBegin()`, `glVertex3f()`, `glEnd()`, `GL_MODELVIEW`, `GL_TEXTURE_2D`, `GL_QUADS`
  - `FileSpecifier`, `ImageDescriptor`, `ImageLoader_Colors` ΓÇö image loading framework
  - `OGL_Blitter` ΓÇö 2D rendering utility

# Source_Files/RenderOther/OGL_LoadScreen.h
## File Purpose
Defines a singleton OpenGL-based load screen manager that displays images with optional progress indicators during loading phases. Handles image rendering, scaling, positioning, and color palette management for loading UIs.

## Core Responsibilities
- Singleton instance lifecycle (Start/Stop)
- Load and display images (full-screen or custom-positioned)
- Update and render progress percentage
- Manage color palette for progress indicator
- Handle image stretching and offset scaling
- Manage OpenGL texture resources

## External Dependencies
- **OpenGL**: Conditional includes (`<OpenGL/gl.h>` macOS, `<GL/gl.h>` Linux/Windows)
- **OGL_Blitter**: Image blitting and texture tile management
- **ImageLoader.h**: `ImageDescriptor` class for loaded image data
- **SDL**: `SDL_Rect` for screen rectangles
- **cseries.h**: Core types (`rgb_color`, `uint32`, vector)

# Source_Files/RenderOther/overhead_map.cpp
## File Purpose
Manages overhead map display configuration and XML parsing for the Aleph One game engine. Delegates rendering to platform-specific implementations (software via SDL/Quickdraw or hardware via OpenGL), and maintains global visual parameters for how the automap displays geometry, entities, and annotations.

## Core Responsibilities
- Maintain global configuration data structure (`OvhdMap_ConfigData`) with all colors, fonts, line/polygon/thing definitions, and display flags
- Parse XML configuration for overhead map appearance, monster/entity display mappings, and rendering modes
- Initialize map fonts on first use
- Select and dispatch to appropriate renderer (OpenGL or software)
- Manage automap visibility reset based on rendering mode (cumulative, currently-visible-only, or all-visible)
- Export XML parser hierarchy for integration with game's configuration system

## External Dependencies
- **Includes**: `cseries.h` (cross-platform utilities), `shell.h` (_get_player_color), `map.h` (world geometry, automap bitsets), `monsters.h` (NUMBER_OF_MONSTER_TYPES), `overhead_map.h` (public interface), `player.h` (player info), `render.h` (rendering integration), `flood_map.h` (pathfinding types), `platforms.h` (platform data), `media.h` (media types), `ColorParser.h` (RGB color parsing)
- **Platform-conditional includes**: `OverheadMap_QD.h` (Mac Quickdraw renderer), `OverheadMap_SDL.h` (SDL software renderer), `OverheadMap_OGL.h` (OpenGL renderer)
- **External symbols**: `dynamic_world` (world state for line/polygon counts), `automap_lines`, `automap_polygons` (visibility bitsets), `Color_GetParser()`, `Font_GetParser()` (color and font XML parsers), renderer classes (`OverheadMapClass`, `OverheadMap_SDL_Class`, etc.)

# Source_Files/RenderOther/overhead_map.h
## File Purpose
Header file defining the overhead map rendering system for the Aleph One game engine. Declares the data structure, rendering function, and XML configuration parser for in-game overhead/mini-map display across multiple contexts (gameplay, saved game preview, checkpoint view).

## Core Responsibilities
- Define constants for overhead map scale constraints
- Declare the `overhead_map_data` struct to hold map rendering state and parameters
- Provide the main overhead map rendering entry point
- Supply XML parsing interface for overhead map configuration elements
- Support multiple rendering modes (saved game, checkpoint, live game)

## External Dependencies
- **`XML_ElementParser.h`** ΓÇô Base class for XML element parsing; supports dynamic configuration of overhead map parameters
- **Implicit dependencies** (defined elsewhere):
  - `world_point2d` ΓÇô 2D world coordinate type
  - World/polygon data structures for map rendering


# Source_Files/RenderOther/OverheadMap_OGL.cpp
## File Purpose
OpenGL renderer implementation for Marathon's overhead map display. Optimizes map drawing via vertex caching and batch rendering to improve performance at high resolutions, replacing CPU-intensive software rendering.

## Core Responsibilities
- Manage OpenGL state setup/teardown for overhead map rendering
- Cache and batch-render polygons, lines, and paths to reduce draw calls
- Render map primitives: polygons, lines, paths, player symbols, and text
- Handle color and pen-width state changes to minimize OpenGL state switches
- Transform and draw game objects (players, monsters, items) on the map overlay

## External Dependencies
- **OpenGL:** `glColor3f`, `glBegin`/`glEnd`, `glVertex*`, `glClear`, `glLineWidth`, `glMatrixMode`, `glPushMatrix`/`glPopMatrix`, `glTranslatef`, `glRotatef`, `glScalef`, `glVertexPointer`, `glDrawArrays`, `glDrawElements`, `glEnable`/`glDisable`, `glEnableClientState`/`glDisableClientState`
- **Game types/functions (defined elsewhere):** `world_point2d`, `world_point3d`, `rgb_color`, `angle`, `GetVertex`, `GetVertexStride`, `GetFirstVertex`, `FontSpecifier::TextWidth`, `FontSpecifier::OGL_Render`, `FULL_CIRCLE`
- **Utilities:** `SglColor3usv` (color wrapper from OGL_Setup.h); `SetColor`, `ColorsEqual` (inline helpers)
- **Platform:** `RenderContext` (Mac AGL context), `ViewWidth`/`ViewHeight` globals

# Source_Files/RenderOther/OverheadMap_OGL.h
## File Purpose
OpenGL-specific implementation of the overhead map renderer. Subclass of `OverheadMapClass` that overrides virtual rendering methods to draw map geometry, entities, and annotations using OpenGL. Provides caching mechanisms for efficient batched rendering of polygons and lines.

## Core Responsibilities
- Override base-class virtual methods to implement OpenGL rendering
- Manage cached polygons and lines for batched GPU submission
- Transform and render world geometry (map polygons, walls, platforms)
- Draw game entities (players, monsters, items) and their paths
- Render text annotations and map labels with font support
- Buffer and flush cached drawing commands in `DrawCached*()` methods

## External Dependencies
- `OverheadMapRenderer.h` ΓÇô base class `OverheadMapClass`, configuration struct `OvhdMap_CfgDataStruct`, type definitions (`rgb_color`, `world_point2d`, `FontSpecifier`)
- `config.h` ΓÇô compile-time guard `HAVE_OPENGL`
- `<vector>` ΓÇô STL container for caching indices and points

# Source_Files/RenderOther/OverheadMap_QD.cpp
## File Purpose
Implementation of a Quickdraw-based overhead map renderer for the Aleph One game engine. Provides concrete drawing primitives (polygons, lines, objects, player icon, text, paths) for the 2D overhead/minimap display on Classic MacOS systems. Subclasses `OverheadMapClass` to replace platform-agnostic rendering calls with Quickdraw-specific graphics API invocations.

## Core Responsibilities
- Render filled polygons (map geometry/walls) with color and border
- Draw line segments between pairs of vertices
- Render game objects (entities) as rectangles or circles with size/color
- Draw player avatar as a directional triangle with rear corners indicating orientation
- Render text labels with left or center justification at map coordinates
- Manage path trace drawing (initialize pen, draw connected line segments)
- Abstract Quickdraw color setup and pen configuration

## External Dependencies
- **Quickdraw API:** `RGBForeColor()`, `OpenPoly()`, `ClosePoly()`, `KillPoly()`, `MoveTo()`, `LineTo()`, `PenSize()`, `PaintRect()`, `FrameOval()`, `FillPoly()`, `FramePoly()`, `SetRect()`, `GetQDGlobalsBlack()`, `DrawString()`, `StringWidth()`, `CopyCStringToPascal()`
- **Parent class:** `OverheadMapClass` (via `OverheadMapRenderer.h`); provides `GetVertex(short)`, `translate_point2d()`, `normalize_angle()`
- **Types defined elsewhere:** `rgb_color`, `world_point2d`, `angle`, `FontSpecifier`
- **C Standard Library:** `<string.h>` for `strncpy()`

# Source_Files/RenderOther/OverheadMap_QD.h
## File Purpose
QuickDraw-specific concrete implementation of the overhead map renderer. Subclasses `OverheadMapClass` to provide Classic MacOS Quickdraw graphics API bindings for rendering minimap/automap features. Authored by Loren Petrich (August 2000) as part of the Aleph One engine.

## Core Responsibilities
- Override virtual rendering methods to use QuickDraw API primitives
- Render map polygons (terrain/walls) with fill colors
- Draw line primitives for walls and elevation changes
- Render entities and items as circles or rectangles
- Draw player avatar with facing direction indicator
- Render text annotations and map names with font support
- Handle path visualization for player movement tracking

## External Dependencies
- **Base class**: `OverheadMapClass` (from `OverheadMapRenderer.h`)
- **Types used (defined elsewhere)**: `rgb_color`, `world_point2d`, `angle`, `FontSpecifier`
- **Included header**: `OverheadMapRenderer.h` (provides base class and enum/struct definitions)

# Source_Files/RenderOther/OverheadMap_SDL.cpp
## File Purpose
SDL-based concrete implementation of the OverheadMapClass for rendering game map elements (polygons, objects, players, paths, text) to the overhead map display. Provides the rendering backend for the Aleph One minimap/automap feature.

## Core Responsibilities
- Render filled/outlined polygons on the overhead map
- Draw line segments connecting map vertices
- Display game objects (scenery/items) as rectangles or octagons
- Render player position and facing direction as directional triangles
- Draw text annotations on the map
- Incrementally render player/unit paths as line chains
- Convert game world colors to SDL pixel values

## External Dependencies
- **SDL library:** `SDL_Surface`, `SDL_Rect`, `SDL_MapRGB()`, `SDL_FillRect()`
- **Global symbols (defined elsewhere):** 
  - `draw_surface` (from screen_sdl.cpp)
  - `::draw_polygon()`, `::draw_line()`, `::draw_text()` (from screen_drawing module)
  - `GetVertex()` (inherited from OverheadMapClass base)
  - `translate_point2d()`, `normalize_angle()` (geometry utilities)
  - `text_width()` (font utility from sdl_fonts)

# Source_Files/RenderOther/OverheadMap_SDL.h
## File Purpose
SDL-specific implementation of the overhead map renderer. This subclass specializes `OverheadMapClass` to draw map elements (polygons, lines, entities, annotations) using SDL graphics primitives. Part of the Aleph One Marathon engine's UI subsystem.

## Core Responsibilities
- Override virtual drawing methods for SDL rendering backend
- Render polygonal map regions with color
- Draw line segments (walls/edges) with configurable pen sizes
- Render map entities (monsters, items, projectiles) as shapes
- Draw player character with direction indicator
- Render text annotations and labels
- Manage path visualization state

## External Dependencies
- **OverheadMapRenderer.h** ΓÇô base class `OverheadMapClass` with virtual interface and configuration data structures
- **SDL** ΓÇô graphics library (linked at runtime, not directly visible in headers)
- Implicit: `world.h` (types: `world_point2d`, `angle`), `FontHandler.h` (`FontSpecifier`)

# Source_Files/RenderOther/OverheadMapRenderer.cpp
## File Purpose
Implements the overhead map renderer for the Aleph One game engine, displaying a top-down view of the game world. Renders polygons, lines, entities, annotations, and paths on the map; handles special checkpoint map mode by generating a temporary view of unexplored areas.

## Core Responsibilities
- Render polygons with color coding based on type (platforms, water, lava, sewage, goo, hills, damage zones)
- Draw map lines with elevation-change and solid-wall coloring
- Transform world coordinates to screen space and determine visibility
- Render game entities (players, monsters, items, projectiles) with team/type colors
- Display map annotations and level names
- Visualize pathfinding paths for debugging
- Generate and manage "false automap" state for checkpoint map rendering (revealing only accessible areas)
- Apply configuration-based display toggles (show aliens/items/projectiles/paths)

## External Dependencies
- **Includes:** `cseries.h` (core types), `OverheadMapRenderer.h` (class and config struct), `flood_map.h` (flood_map function), `media.h` (media_data struct), `platforms.h` (platform_data, macros), `player.h` (player_data, monster_index_to_player_index), `render.h` (render flags, view_data), `string.h`, `stdlib.h`, `limits.h`.
- **External Symbols:**
  - `dynamic_world` (current world state)
  - `static_world` (static data)
  - `objects` (game entity array)
  - `saved_objects` (checkpoint objects)
  - `automap_lines`, `automap_polygons` (visibility state)
  - `local_player` (player data)
  - `get_polygon_data()`, `get_line_data()`, `get_media_data()`, `get_endpoint_data()`, `get_platform_data()`, `get_player_data()`, `get_monster_data()` (accessors)
  - `get_next_map_annotation()`, `path_peek()`, `GetNumberOfPaths()` (iteration/lookup)
  - `monster_index_to_player_index()` (conversion)
  - `flood_map()` (pathfinding/flood fill)
  - Macros: `POLYGON_IS_IN_AUTOMAP()`, `TEST_STATE_FLAG()`, `SET_STATE_FLAG()`, `LINE_IS_IN_AUTOMAP()`, `LINE_IS_SOLID()`, `LINE_IS_VARIABLE_ELEVATION()`, `LINE_IS_LANDSCAPED()`, `PLATFORM_IS_SECRET()`, `PLATFORM_IS_DOOR()`, `POLYGON_IS_DETACHED()`, `OBJECT_IS_INVISIBLE()`, `GET_OBJECT_OWNER()`, `MONSTER_IS_PLAYER()`, `SLOT_IS_USED()`, `GET_GAME_OPTIONS()`, `WORLD_TO_SCREEN()`, `WORLD_ONE`, `NONE`, `UNONE`, `ADD_LINE_TO_AUTOMAP()`, `ADD_POLYGON_TO_AUTOMAP()`.

# Source_Files/RenderOther/OverheadMapRenderer.h
## File Purpose
Defines the base class and configuration structures for rendering overhead (automap) displays in the game engine. Provides a virtual interface for graphics-API-agnostic map rendering, allowing subclasses to implement rendering via OpenGL, software rasterization, or other backends.

## Core Responsibilities
- Declare configuration structures for map visual properties (colors, fonts, shapes, line styles)
- Define enums for polygon types, line types, and entity types used in overhead maps
- Provide `OverheadMapClass` base class with virtual methods for rendering map elements
- Support both real automaps (explicit polygon/line data) and "false" automaps (generated via pathfinding)
- Manage player entity shape representations and directional indicators on the map

## External Dependencies

- **cseries.h** ΓÇô Core types and utilities
- **world.h** ΓÇô `world_point2d`, `world_point3d`, `angle`, world coordinate macros
- **map.h** ΓÇô Map geometry (`endpoint_data`, line/polygon accessors)
- **monsters.h** ΓÇô Monster type constants (NUMBER_OF_MONSTER_TYPES)
- **overhead_map.h** ΓÇô `overhead_map_data` struct, scale/mode constants (OVERHEAD_MAP_MINIMUM_SCALE, etc.)
- **shape_descriptors.h** ΓÇô `shape_descriptor` type
- **shell.h** ΓÇô `_get_player_color()` function, RGBColor type
- **FontHandler.h** ΓÇô `FontSpecifier` class for text rendering

**External symbols used:**
- `get_endpoint_data(short index)` ΓÇô Returns endpoint_data for a given vertex (map.h)
- `get_line_data(short line_index)` ΓÇô Returns line_data struct (map.h)
- `_get_player_color(size_t color_index, RGBColor*)` ΓÇô Maps player color index to RGB (shell.h)

# Source_Files/RenderOther/screen.cpp
## File Purpose
Manages SDL-based screen rendering, display modes, and framebuffer allocation for the Aleph One game engine. Handles screen initialization, per-frame rendering orchestration, color tables, OpenGL context setup, and HUD/terminal display.

## Core Responsibilities
- Initialize and switch display modes (windowed/fullscreen, resolution, bit depth, OpenGL)
- Allocate and manage SDL rendering surfaces (world_pixels, HUD_Buffer, Term_Buffer)
- Manage color tables and gamma correction
- Orchestrate per-frame rendering pipeline (viewport ΓåÆ world view ΓåÆ HUD ΓåÆ terminal ΓåÆ screen blit)
- Calculate and maintain view rectangles for world, HUD, overhead map, and terminal
- Handle screen clearing and margin filling
- Provide screen dimension and mode accessors to other subsystems

## External Dependencies
- **SDL headers**: `<SDL.h>`, `<SDL_opengl.h>` (conditional)
- **Game engine headers**: world.h, map.h, render.h, shell.h, interface.h, player.h, preferences.h, game_window.h, overhead_map.h, fades.h, computer_interface.h, ViewControl.h
- **Rendering**: OGL_Blitter.h, OGL_Render.h, screen_drawing.h
- **Scripting**: lua_script.h, lua_hud_script.h, HUDRenderer_Lua.h
- **Utilities**: cseries.h (SDL, math, ctype, stdlib, string, algorithm)
- **External symbols used**: render_view(), initialize_view_data(), ChaseCam_GetPosition(), OGL_IsActive(), OGL_SetWindow(), OGL_StartRun(), OGL_StopRun(), OGL_RenderCrosshairs(), L_Call_HUDResize(), Lua_DrawHUD(), OGL_DrawHUD(), SDL_*() family, glGetString(), glScissor(), glViewport()

# Source_Files/RenderOther/screen.h
## File Purpose
Header for the game engine's screen and display management system. Defines a singleton `Screen` class for managing resolution, rendering surfaces, HUD layout, and color tables. Declares functions for rendering, effects (gamma, tunnel vision, teleporting), and overlay HUD elements.

## Core Responsibilities
- Screen mode enumeration and selection (resolution management)
- Geometry calculation for 3D view, HUD, map, and terminal rendering areas
- Color table (palette) management and CLUT (Color Look-Up Table) animation
- Visual effects (teleporting, extravision, tunnel vision, gamma correction)
- Screen dumping (screenshots) and buffer management
- Overhead map display control and zoom
- Script-controlled HUD element rendering (text, icons, colored squares)
- OpenGL acceleration mode support

## External Dependencies
- Standard library: `<utility>`, `<vector>`
- SDL types: `SDL_Rect` (for rectangles)
- Forward-declared: `screen_mode_data` struct (defined elsewhere)
- Extern globals: `color_table` structures for world, visible, and interface rendering
- Notes: Code predates C++11 (uses `std::pair<int, int>` with old space syntax); copyright 1991ΓÇô2001, Bungie/Aleph One project.

# Source_Files/RenderOther/screen_definitions.h
## File Purpose
Centralized enum defining base resource IDs for game screen types (intro, menu, prologue, credits, etc.). Used to map screen types to PICT image resource ranges across 8-bit, 16-bit, and 32-bit color depths.

## Core Responsibilities
- Define base resource ID constants for each screen type
- Provide offset mapping for bitdepth variants (8-bit + 0, 16-bit + 10000, 32-bit + 20000)
- Establish a naming convention for screen resource organization

## External Dependencies
None.

# Source_Files/RenderOther/screen_drawing.cpp
## File Purpose
Low-level SDL-based screen drawing implementation for the Aleph One game engine. Provides services for rendering shapes, text, geometric primitives, and managing the drawing surface, clipping, colors, and fonts for both in-game HUD and menus.

## Core Responsibilities
- Manage drawing surfaces (screen, world buffer, HUD buffer, terminal buffer) via port switching
- Render 2D shapes and sprites with coordinate translation and clipping
- Render text using bitmap fonts (sdl_font_info) and TrueType fonts (ttf_font_info)
- Draw geometric primitives: filled/outlined rectangles, lines (with Cohen-Sutherland clipping), and filled convex polygons
- Maintain interface rectangle definitions, color palettes, and font specifications
- Parse interface rectangles and colors from XML configuration
- Support text layout: wrapping, horizontal/vertical centering, right/top justification

## External Dependencies
- **SDL library**: `SDL_Surface`, `SDL_Rect`, `SDL_BlitSurface`, `SDL_FillRect`, `SDL_UpdateRects`, `SDL_GetVideoSurface`, `SDL_LockSurface`, `SDL_UnlockSurface`, `SDL_DisplayFormat`, `SDL_MapRGB`, `SDL_FreeSurface`
- **SDL_TTF**: `TTF_RenderUTF8_Blended`, `TTF_RenderUTF8_Solid`, `TTF_RenderUNICODE_Blended`, `TTF_RenderUNICODE_Solid`, `TTF_FontAscent`, `TTF_FontHeight`
- **Game engine headers**:
  - `shape_descriptors.h` ΓÇö shape descriptor decoding
  - `sdl_fonts.h` ΓÇö font abstraction (`sdl_font_info`, `ttf_font_info`, `font_info` base)
  - `ColorParser.h`, `FontHandler.h` ΓÇö XML configuration support
  - `map.h`, `interface.h`, `shell.h`, `screen.h` ΓÇö game state and callbacks
  - `fades.h` ΓÇö screen effects (included but not directly used in this file)

# Source_Files/RenderOther/screen_drawing.h
## File Purpose
Header file defining the HUD/UI rendering interface for a Marathon-like first-person shooter. Provides screen drawing primitives, text rendering, and shape display functions, along with enumerated constants for UI rectangles, colors, fonts, and text justification options. Supports both legacy and SDL rendering backends.

## Core Responsibilities
- Define UI layout constants (rectangle IDs for HUD elements, buttons)
- Manage rendering target surfaces (screen window, offscreen buffer, HUD buffer, terminal)
- Provide high-level drawing functions for shapes, text, and rectangles
- Expose text measurement utilities for layout and wrapping
- Parse XML configuration for interface rectangle and color definitions
- Supply font and color palette management for UI rendering
- Offer SDL-specific inline wrappers for text and primitive drawing

## External Dependencies
- **XML_ElementParser.h**: XML parsing framework for dynamic interface configuration
- **shape_descriptors.h**: Shape ID encoding (collection, CLUT, shape bits)
- **sdl_fonts.h**: Font info class and SDL font loading
- **SDL (conditional)**: `SDL_Surface`, `SDL_Rect`, `SDL_ttf` for rendering and text drawing
- **Implicit platform abstractions**: Port/GWorld concepts (classic Mac Toolbox patterns)

# Source_Files/RenderOther/screen_shared.h
## File Purpose
Shared header between screen rendering implementations (screen.cpp and screen_sdl.cpp). Defines screen dimensions, global color/view state, screen messaging, script HUD elements, and core display functions for rendering the game world, HUD, text overlays, and network status information.

## Core Responsibilities
- Define screen resolution constants for various platforms (Dingoo, standard)
- Manage global color tables (uncorrected, gamma-corrected, interface, visible)
- Maintain screen mode and field-of-view state
- Implement frame-rate display, position debugging, and message queue system
- Manage script-driven HUD elements (icons, text, colors)
- Provide on-screen text rendering (FPS, position, messages, network scores/mic status)
- Handle screen reset, zoom, and effect transitions (teleport, extravision, tunnel vision)
- Coordinate HUD and terminal rendering requests

## External Dependencies
- **Config & string handling:** `config.h`, `<stdarg.h>`, `snprintf.h` (platform compatibility)
- **Rendering:** `screen_drawing.h`, `Image_Blitter.h`, `OGL_Blitter.h` (blitter/image loading)
- **Networking:** `network_games.h` (game state, player rankings, net stats)
- **Console/UI:** `Console.h` (input state for overlay positioning)
- **Symbols defined elsewhere:** `uncorrected_color_table`, `world_color_table`, `interface_color_table`, `visible_color_table`, `world_view`, `world_pixels_structure`, `current_player`, `dynamic_world`, `current_player_index`, `player_in_terminal_mode()`, `View_DoFoldEffect()`, `start_render_effect()`, `OGL_MapActive`, `OGL_IsActive()`, `OGL_RenderText()`, `gamma_correct_color_table()`, `stop_fade()`, `obj_copy()`, `assert_world_color_table()`, `change_screen_mode()`, `set_fade_effect()`, `GetOnScreenFont()`, `SDL_MapRGB()`, `NetGetLatency()`, `current_netgame_allows_microphone()`, `get_player_data()`, `_get_interface_color()`, `_computer_interface_text_color`, `get_screen_mode()`, `alephone::Screen::instance()`, `local_player_index`, `game_is_networked`, `temporary` (global buffer), `MACHINE_TICKS_PER_SECOND`, `TICKS_PER_SECOND`, `dirty_terminal_view()`.

# Source_Files/RenderOther/sdl_fonts.cpp
## File Purpose
Manages font loading, caching, and text measurement for the Aleph One engine. Supports both bitmap fonts (Mac-style resource format) and TrueType fonts via SDL_ttf, with reference counting and styled text parsing (bold, italic, plain).

## Core Responsibilities
- Initialize font resources from disk (bitmap "Fonts" resources or TTF files)
- Load and cache bitmap fonts (`sdl_font_info`) with reference counting
- Load and cache TrueType fonts (`ttf_font_info`) with multiple style variants (normal, bold, italic, bold+italic)
- Parse and apply inline style codes (`|b`, `|i`, `|p`) in text strings
- Measure text width considering font metrics, style, and shadow effects
- Truncate text to fit within max width constraints
- Handle Mac Roman Γåö Unicode encoding for TTF text
- Unload fonts with safe deallocation via reference counting

## External Dependencies
- **SDL:** `<SDL_endian.h>` (byte-order I/O), SDL_RWops, SDL_ttf (TTF_Font, TTF_GlyphMetrics, TTF_SizeUTF8/UNICODE, TTF_OpenFont, TTF_SetFontStyle, TTF_SetFontHinting, TTF_CloseFont)
- **Boost:** `<boost/tokenizer.hpp>` (custom tokenizer separator)
- **Engine headers:** `cseries.h`, `sdl_fonts.h` (type defs), `byte_swapping.h`, `resource_manager.h` (get_resource, LoadedResource, open_res_file), `FileHandler.h` (FileSpecifier), `Logging.h` (logContext, logFatal), `preferences.h` (environment_preferences), `AlephSansMono-Bold.h` (embedded font data)
- **Defined elsewhere:** `mac_roman_to_unicode()` (encoding conversion), `_draw_text()` methods (screen_drawing.cpp), `fix_missing_*()` functions, `data_search_path` (shell_sdl.cpp)

# Source_Files/RenderOther/sdl_fonts.h
## File Purpose
Defines the font rendering interface for the Aleph One game engine, supporting both bitmap fonts (`sdl_font_info`) and TrueType fonts (`ttf_font_info`) via SDL. Provides abstractions for text drawing, measurement, and styling across different font backends.

## Core Responsibilities
- Define abstract `font_info` interface for font metrics and text rendering
- Implement bitmap font rendering with kerning and styling support
- Implement TrueType font rendering (conditional on `HAVE_SDL_TTF`)
- Provide text measurement, truncation, and styled text operations
- Manage font resource lifecycle (loading/unloading via `LoadedResource`)
- Support multi-style rendering (bold, italic, underline, etc.)

## External Dependencies
- **FileHandler.h** ΓÇô `LoadedResource` class for resource lifecycle management
- **SDL_ttf.h** (conditional) ΓÇô TTF rendering; `TTF_Font`, `TTF_FontAscent()`, etc.
- **boost/tuple/tuple.hpp** (conditional) ΓÇô `ttf_font_key_t` tuple template
- **string** (std) ΓÇô styled text strings
- Undefined symbols used: `TextSpec` (assumed defined elsewhere in render/UI headers), `uint16`, `uint32`, `int16`, `int8`, `uint8` (fixed-width types)

# Source_Files/RenderOther/Shape_Blitter.cpp
## File Purpose
Renders 2D UI bitmaps from the game's shape resources to screen. Supports dual rendering backends: OpenGL (hardware-accelerated) and SDL software rasterization. Handles shape scaling, rotation, cropping, and tinting.

## Core Responsibilities
- Construct Shape_Blitter objects from collection/texture descriptors
- Lazy-load and cache SDL surfaces for shapes
- Rescale shapes with proportional crop-rectangle adjustment
- Render shapes via OpenGL with texture manager setup and multiple texture-type handling
- Render shapes via SDL with surface transformations (rotation, flip, rescale)
- Apply visual effects: tinting (RGBA), rotation about center, cropping
- Manage surface memory lifecycle (allocation, caching, deallocation)

## External Dependencies
- **Notable includes:**
  - `Shape_Blitter.h` ΓÇô class definition
  - `interface.h` ΓÇô `get_shape_bitmap_and_shading_table`, shape descriptor macros
  - `render.h` ΓÇô rendering structures and transfer modes
  - `images.h` ΓÇô `rescale_surface`, SDL surface utilities
  - `shell.h` ΓÇô `get_shape_surface`
  - `scottish_textures.h` ΓÇô transfer modes, `_shading_normal`, `_shadeless_transfer`, `_tinted_transfer`
  - `OGL_Setup.h` ΓÇô OpenGL configuration, `SglColor4f`, `TextureManager`
  - `OGL_Textures.h` ΓÇô texture manager internals
  - `OGL_Blitter.h` ΓÇô OpenGL blitting utilities
  - `<GL/gl.h>`, `<GL/glu.h>`, `<GL/glext.h>` (conditional on platform and `HAVE_OPENGL`)
  - `<SDL/SDL.h>` ΓÇô SDL surface, rect, and pixel types

- **Defined elsewhere:** `rotate_surface` (declared; implemented in `HUDRenderer_SW.cpp`), `rescale_surface`, `get_shape_surface`, `get_shape_bitmap_and_shading_table`, `View_GetLandscapeOptions`, OpenGL context and matrix state

# Source_Files/RenderOther/Shape_Blitter.h
## File Purpose
Provides a utility class for rendering 2D bitmap images from a Shapes resource file (Marathon-format graphics) to SDL surfaces or OpenGL targets. Handles scaling, tinting, rotation, and cropping of shape bitmaps for UI and texture drawing.

## Core Responsibilities
- Load and manage shape descriptors from the game's resource collections
- Scale shapes to fit arbitrary destination dimensions on demand
- Render shapes via both SDL (CPU) and OpenGL (GPU) paths
- Apply visual effects: color tinting, rotation about center, and rectangular cropping
- Manage SDL surface memory and lifecycle for scaled versions

## External Dependencies
- **cseries.h**: Base engine types and utilities
- **map.h**: shape_descriptor type; shape resource system
- **SDL/SDL.h**: SDL_Rect, SDL_Surface, rendering primitives
- **\<vector\>, \<set\>**: STL containers (included but usage not visible in header)
- **shape_descriptors.h**: Shape descriptor type definitions (transitively via map.h)

# Source_Files/RenderOther/TextLayoutHelper.cpp
## File Purpose
Implements a rectangle layout helper that tracks and reserves non-overlapping rectangular regions in 2D space. It provides functionality to position new rectangles without colliding with existing reservations, primarily for UI text/element placement.

## Core Responsibilities
- Maintain a sorted collection of rectangle boundaries indexed by horizontal coordinate
- Track which reservations overlap in X-space with newly positioned rectangles
- Calculate the lowest valid Y-position for a new rectangle given constraints
- Manage memory for dynamically allocated reservation objects
- Iterate vertically upward until finding a non-overlapping position

## External Dependencies
- `<vector>` ΓÇö for `CollectionOfReservationEnds` storage
- `<set>` ΓÇö for temporary `multiset<Reservation*>` during overlap detection
- `<assert.h>` ΓÇö assertions on input height validity
- `TextLayoutHelper.h` ΓÇö class definition and nested struct declarations

# Source_Files/RenderOther/TextLayoutHelper.h
## File Purpose
Defines a utility class for managing non-overlapping rectangular reservations in a 2D space. Used by the Marathon: Aleph One game engine to calculate optimal placement positions for text or UI elements without overlap.

## Core Responsibilities
- Reserve rectangular space with automatic collision avoidance
- Calculate optimal vertical placement given horizontal bounds and desired height
- Maintain a collection of active reservations tracked by endpoint coordinates
- Clear all reservations for reuse

## External Dependencies
- `#include <vector>` ΓÇö STL vector for dynamic reservation tracking
- `using namespace std` ΓÇö Brings std namespace into scope

# Source_Files/RenderOther/TextStrings.cpp
## File Purpose
Implements a text string collection system that replaces MacOS STR# resources with a portable, XML-loadable string repository. Strings are organized by ID (resource-like) and index, stored as MacOS Pascal strings (length byte + chars), and can be retrieved as either Pascal or C strings.

## Core Responsibilities
- Manage multiple string collections (sets) indexed by ID using a linked list
- Provide Pascal and C string storage/retrieval operations
- Dynamically grow string arrays on demand (doubling strategy)
- Load strings from XML documents via callback-based parser
- Handle cleanup and deletion at granular (string, set) and bulk (all sets) levels

## External Dependencies
- **Includes**: `<string.h>`, `cseries.h` (SDL, MacOS type shims), `TextStrings.h` (own header), `XML_ElementParser.h` (XML framework)
- **Helper functions** (defined elsewhere): `objlist_clear()`, `objlist_copy()`, `StringsEqual()`, `ReadInt16Value()`, `DeUTF8_Pas()` (XML_ElementParser.h)
- **MacOS types**: `Str255` (256-byte Pascal string buffer), `int16`, `uint16`, `size_t`

---

**Notes**: 
- Code is O(n) in number of string sets; acceptable for small counts but could optimize with hash table
- Repeated linked-list traversal patterns are not DRY
- Signed/unsigned mismatch in `Index < 0` conditions (index is `size_t`)
- No thread safety; assumes single-threaded access

# Source_Files/RenderOther/TextStrings.h
## File Purpose
Header file declaring a text-string repository interface that replaces legacy MacOS STR# resources. Provides functions to store, retrieve, and manage strings organized by resource ID and index, supporting both Pascal and C string formats.

## Core Responsibilities
- Store and retrieve strings in Pascal (length-byte prefix) and C (null-terminated) formats
- Manage string collections grouped by resource ID
- Query existence and count of string sets
- Delete individual strings or entire string sets
- Provide XML parsing interface for string data loading

## External Dependencies
- `XML_ElementParser` class (defined elsewhere; likely in an XML parsing subsystem)
- No standard library includes visible here (implementation likely includes stdio, memory utilities)

# Source_Files/RenderOther/ViewControl.cpp
## File Purpose
Manages camera/view control settings for the game engine, including field-of-view parameters, teleportation visual effects, landscape texture options, and on-screen rendering configuration. Provides XML-based configuration system for customizing view behavior.

## Core Responsibilities
- Maintain FOV settings (normal, extra vision, tunnel vision) and smooth FOV transitions
- Control optional visual/audio effects for teleportation (folding, static, interlevel effects)
- Store and retrieve landscape texture options indexed by collection/frame pairs
- Parse and apply view-related settings from XML configuration
- Manage on-screen display font initialization and access

## External Dependencies
- **Includes:** `<vector>`, `<string.h>` (STL); `"cseries.h"` (macros, types); `"world.h"` (angle, shape descriptor macros); `"shell.h"`, `"screen.h"`, `"ViewControl.h"`, `"SoundManager.h"` (interfaces)
- **Used elsewhere:** `StringsEqual()`, `ReadBoundedNumericalValue()`, `ReadBooleanValueAsBool()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `ReadInt16Value()` (XML parsing utilities); `get_screen_mode()` (screen mode query); `Font_SetArray()`, `Font_GetParser()` (font system); shape descriptor macros (`GET_DESCRIPTOR_SHAPE`, `GET_COLLECTION`, `MAXIMUM_SHAPES_PER_COLLECTION`); angle constants (`FULL_CIRCLE`)
- **Defined elsewhere:** `XML_ElementParser` base class, `FontSpecifier`, `LandscapeOptions` (declared in ViewControl.h header)

# Source_Files/RenderOther/ViewControl.h
## File Purpose
View controller module for the Marathon/Aleph One game engine. Controls viewing parameters including field-of-view, landscape rendering options, teleport effects, and on-screen display settings. Provides XML-based configuration support for customizing these visual parameters.

## Core Responsibilities
- Manage field-of-view (FOV) states: normal, extravision, and tunnel vision modes
- Provide smooth FOV adjustments toward target values
- Control visual effects for teleportation (fold effect, static effect, interlevel effects)
- Query and configure landscape rendering parameters
- Provide on-screen display font access
- Manage XML-based configuration for all view settings
- Check overhead map availability status

## External Dependencies
**Includes:**
- `world.h` ΓÇô World coordinates, angle type definition
- `FontHandler.h` ΓÇô FontSpecifier class
- `shape_descriptors.h` ΓÇô shape_descriptor typedef and collection/clut macros
- `XML_ElementParser.h` ΓÇô XML parsing framework

**External symbols used but not defined here:**
- `FontSpecifier` (class)
- `shape_descriptor` (typedef'd as uint16)
- `angle` (typedef'd as int16)
- `XML_ElementParser` (class)


