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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Screen | class (singleton) | Main display management interface |
| screen_mode_data | struct | Display configuration (resolution, depth, acceleration, scaling) |
| view_data | struct | Camera/viewport configuration (FOV, screen size, origin, effects) |
| bitmap_definition | struct | Drawing buffer metadata (width, height, pixel data pointers) |
| color_table | struct | Palette data (256 colors with 16-bit RGB) |
| SDL_Surface | extern type | SDL framebuffer (main_surface, world_pixels, HUD_Buffer, Term_Buffer) |
| OGL_Blitter | class | OpenGL texture blitter for terminal rendering |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| main_surface | SDL_Surface* | static | The actual display framebuffer |
| world_pixels | SDL_Surface* | file-static | Off-screen buffer for world view rendering |
| HUD_Buffer | SDL_Surface* | file-static | Off-screen buffer for HUD rendering |
| Term_Buffer | SDL_Surface* | file-static | Off-screen buffer for terminal rendering |
| default_gamma_r/g/b | uint16[256] | static | Default gamma correction tables |
| screen_initialized | bool | static | Has screen been initialized |
| in_game | bool | static | Are we in-game (vs. menu) |
| desktop_width/height | static int | static | Native desktop dimensions |
| prev_width/height | static int | static | Previous screen dimensions (for HUD resize detection) |
| world_color_table, visible_color_table, interface_color_table | color_table* | static | Active color palettes |
| Term_Blitter | OGL_Blitter | static | OpenGL blitter for rendering terminal to screen |

## Key Functions / Methods

### Screen::Initialize
- **Signature**: `void Initialize(screen_mode_data* mode)`
- **Purpose**: One-time initialization of screen subsystem; allocates color tables, view data, bitmap structures; enumerates available display modes.
- **Inputs**: Display mode configuration (resolution, bit depth, acceleration)
- **Outputs/Return**: None; initializes global screen state
- **Side effects**: Allocates global structs (color tables, world_view, world_pixels_structure); calls change_screen_mode(); sets screen_initialized flag
- **Calls**: change_screen_mode(), malloc(), memset(), SDL_ListModes(), SDL_GetVideoInfo()
- **Notes**: Handles both Dingoo (fixed 320├ù240) and standard platforms with fallback mode enumeration; skipped on re-entry

### enter_screen
- **Signature**: `void enter_screen(void)`
- **Purpose**: Transition from menu to in-game; set up game resolution and reset state.
- **Inputs**: None (reads global screen_mode)
- **Outputs/Return**: None
- **Side effects**: Sets in_game flag; calls change_screen_mode(); resets view effects; initializes Lua HUD rectangles; calls L_Call_HUDResize()
- **Calls**: set_overhead_map_status(), set_terminal_status(), change_screen_mode(), OGL_StartRun(), SDL_SetModState()
- **Notes**: Clears overhead map and terminal mode before switching

### exit_screen
- **Signature**: `void exit_screen(void)`
- **Purpose**: Transition from in-game back to menu (fixed 640├ù480, no OpenGL).
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Sets in_game=false; reverts to fixed menu resolution; calls OGL_StopRun() if OpenGL active
- **Calls**: change_screen_mode(), OGL_StopRun()

### change_screen_mode (overload 1: int,int,int,bool)
- **Signature**: `static void change_screen_mode(int width, int height, int depth, bool nogl)`
- **Purpose**: Low-level SDL display mode switch; allocates OpenGL context or software surface.
- **Inputs**: Target width, height, bit depth; nogl=true disables OpenGL
- **Outputs/Return**: None
- **Side effects**: Calls SDL_SetVideoMode(); sets up OpenGL attributes or SDL surface flags; clears HUD/Term buffers; prints GL info
- **Calls**: SDL_SetVideoMode(), SDL_GL_SetAttribute(), glScissor(), glViewport(), clear_screen()
- **Notes**: Retries with 16-bit color if 24-bit OpenGL fails; desktop dims used if fullscreen & !fill_the_screen; hardcoded to 320├ù240 on Dingoo

### change_screen_mode (overload 2: screen_mode_data*, bool)
- **Signature**: `void change_screen_mode(struct screen_mode_data *mode, bool redraw)`
- **Purpose**: High-level display mode switch; updates world buffers, view geometry, and renders if needed.
- **Inputs**: New screen_mode; redraw flag
- **Outputs/Return**: None
- **Side effects**: Copies mode to global screen_mode; calls low-level change_screen_mode(); recalculates view rectangles; reallocates world_pixels; calls render_view(); updates screen
- **Calls**: Low-level change_screen_mode(), initialize_view_data(), reallocate_world_pixels(), render_view(), update_screen(), OGL_SetWindow()
- **Notes**: Only reallocates buffers if size changed; clears screen on size change or when clear_next_screen flag set

### render_view_in_frame
- **Signature**: `void render_view_in_frame(int ticks_elapsed)`
- **Purpose**: Main per-frame rendering; orchestrates all render passes (world, HUD, terminal).
- **Inputs**: Elapsed ticks since last frame
- **Outputs/Return**: None
- **Side effects**: Renders world view, crosshairs, FPS/position/scores/messages, HUD, terminal; updates screen via SDL or OpenGL
- **Calls**: render_view(), Crosshairs_Render(), update_fps_display(), DisplayPosition(), DisplayMessages(), DisplayNetMicStatus(), DisplayScores(), DisplayInputLine(), Lua_DrawHUD(), OGL_DrawHUD(), DrawSurface(), update_screen(), OGL_SwapBuffers(), SDL_UpdateRect()
- **Notes**: Switches between OpenGL and software rendering based on acceleration mode; clears Lua drawing after render on SDL, before on OpenGL; handles terminal rendering separately

### reallocate_world_pixels
- **Signature**: `static void reallocate_world_pixels(int width, int height)`
- **Purpose**: Allocate off-screen buffer for world view rendering.
- **Inputs**: Target width and height
- **Outputs/Return**: None; sets global world_pixels
- **Side effects**: Frees previous world_pixels; creates new SDL_Surface; sets palette if 8-bit
- **Calls**: SDL_FreeSurface(), SDL_CreateRGBSurface(), build_sdl_color_table(), SDL_SetColors()

### Screen accessor methods
- **Signatures**: `int height()`, `int width()`, `int window_height()`, `int window_width()`, `bool hud()`, `bool lua_hud()`, `bool openGL()`, `bool fifty_percent()`, `bool seventyfive_percent()`
- **Purpose**: Query current screen state
- **Notes**: window_* methods clamp to minimum (480 or 240 depending on Dingoo); percent methods check screen_mode.height

### Rectangle calculation methods
- **Signatures**: `SDL_Rect window_rect()`, `SDL_Rect view_rect()`, `SDL_Rect map_rect()`, `SDL_Rect term_rect()`, `SDL_Rect hud_rect()`
- **Purpose**: Calculate layout rectangles for each UI element, accounting for HUD scale, terminal scale, Lua HUD override, and aspect ratio
- **Notes**: view_rect() applies 50% and 75% scaling from Dingoo ifdef (commented out otherwise); term_rect() and hud_rect() respect scale_level from screen_mode

### Color table functions
- **`initialize_gamma()`**: Caches system gamma ramp on first call
- **`restore_gamma()`**: Restores cached gamma (skipped on Dingoo)
- **`build_direct_color_table()`**: Creates RGB color table from gamma
- **`change_screen_clut()`, `change_interface_clut()`, `animate_screen_clut()`, `assert_world_color_table()`**: Update active color palettes and apply to surfaces
- **Notes**: On Dingoo, animate_screen_clut() uses software gamma lookup tables instead of SDL_SetGammaRamp()

### Rendering sub-functions
- **`render_computer_interface()`**: Calls terminal rendering pipeline
- **`render_overhead_map()`**: Renders overhead map to world_pixels
- **`darken_world_window()`**: Draws dithered black overlay (OpenGL or software)
- **`clear_screen()`**: Fills with black (OpenGL ClearColor or SDL_FillRect)
- **`clear_screen_margin()`**: Fills margins around active render area
- **`DrawSurface()`**: Blits HUD/terminal with scaling
- **`update_screen()`**: Blits world_pixels to main_surface, with 2├ù2 pixel quadrupling on low-res
- **`validate_world_window()`**: Marks terminal for redraw

## Control Flow Notes
**Initialization phase:**
1. `Screen::Initialize()` called once at startup ΓåÆ allocates all global state, enumerates modes
2. `change_screen_mode()` sets initial 640├ù480 no-OpenGL menu screen

**Game flow:**
1. `enter_screen()` ΓåÆ switches to game resolution, initializes view data
2. Each frame: `render_view_in_frame()` ΓåÆ renders world, HUD, terminal ΓåÆ updates screen
3. `exit_screen()` ΓåÆ reverts to menu

**Mode changes:**
- During gameplay, `change_screen_mode(screen_mode_data*)` called to resize ΓåÆ reallocates buffers, recalculates layout
- Changes to OpenGL or fullscreen state trigger context reload via `change_screen_mode(int,int,int,bool)`

## External Dependencies
- **SDL headers**: `<SDL.h>`, `<SDL_opengl.h>` (conditional)
- **Game engine headers**: world.h, map.h, render.h, shell.h, interface.h, player.h, preferences.h, game_window.h, overhead_map.h, fades.h, computer_interface.h, ViewControl.h
- **Rendering**: OGL_Blitter.h, OGL_Render.h, screen_drawing.h
- **Scripting**: lua_script.h, lua_hud_script.h, HUDRenderer_Lua.h
- **Utilities**: cseries.h (SDL, math, ctype, stdlib, string, algorithm)
- **External symbols used**: render_view(), initialize_view_data(), ChaseCam_GetPosition(), OGL_IsActive(), OGL_SetWindow(), OGL_StartRun(), OGL_StopRun(), OGL_RenderCrosshairs(), L_Call_HUDResize(), Lua_DrawHUD(), OGL_DrawHUD(), SDL_*() family, glGetString(), glScissor(), glViewport()
