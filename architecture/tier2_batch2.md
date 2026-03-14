# Architecture Overview

## Repository Shape

```
C:\Coding\Bungie\alephone-dingoo/
Γö£ΓöÇΓöÇ Source_Files/
Γöé   Γö£ΓöÇΓöÇ RenderMain/              # 3D-to-2D rendering pipeline
Γöé   Γöé   Γö£ΓöÇΓöÇ render.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ NewRenderVisTree.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ RenderSortPoly.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ RenderPlaceObjs.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ RenderRasterize.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ Rasterizer*.h
Γöé   Γöé   Γö£ΓöÇΓöÇ scottish_textures.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ AnimatedTextures.cpp/h
Γöé   Γöé   ΓööΓöÇΓöÇ [shape/texture/image loaders]
Γöé   Γö£ΓöÇΓöÇ RenderOther/             # Screen, HUD, UI, images, fonts
Γöé   Γöé   Γö£ΓöÇΓöÇ screen.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ screen_drawing.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ game_window.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ HUDRenderer*.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ images.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ [fonts, effects, overlays]
Γöé   Γöé   ΓööΓöÇΓöÇ [automap, computer interface, motion sensor]
Γöé   Γö£ΓöÇΓöÇ Sound/                   # Audio playback and mixing
Γöé   Γöé   Γö£ΓöÇΓöÇ SoundManager.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ Mixer.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ Music.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ *Decoder.cpp/h       # Format decoders (AIFF, MP3, Vorbis, etc.)
Γöé   Γöé   ΓööΓöÇΓöÇ sound_definitions.h
Γöé   Γö£ΓöÇΓöÇ Network/
Γöé   Γöé   ΓööΓöÇΓöÇ TCPMess/             # TCP message framing and routing
Γöé   Γöé       Γö£ΓöÇΓöÇ CommunicationsChannel.cpp/h
Γöé   Γöé       Γö£ΓöÇΓöÇ Message.cpp/h
Γöé   Γöé       ΓööΓöÇΓöÇ MessageDispatcher.cpp/h
Γöé   Γö£ΓöÇΓöÇ XML/                     # Configuration parsing via Expat
Γöé   Γöé   Γö£ΓöÇΓöÇ XML_ElementParser.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ XML_Configure.cpp/h
Γöé   Γöé   Γö£ΓöÇΓöÇ XML_Loader_SDL.cpp/h
Γöé   Γöé   ΓööΓöÇΓöÇ *Parser.cpp/h        # Subsystem-specific parsers
Γöé   ΓööΓöÇΓöÇ [Other subsystems: GameWorld, Input, Files, Misc, CSeries]
Γö£ΓöÇΓöÇ Expat/                        # Bundled Expat XML parser (C library)
Γö£ΓöÇΓöÇ tools/                        # Development utilities
Γöé   Γö£ΓöÇΓöÇ dumprsrcmap.cpp
Γöé   Γö£ΓöÇΓöÇ dumpwad.cpp
Γöé   ΓööΓöÇΓöÇ MapChunkerMain.cpp
Γö£ΓöÇΓöÇ dingux-build/                # MIPS cross-compile build (mipsel-linux-g++)
ΓööΓöÇΓöÇ [configuration, resources]
```

---

## Major Subsystems

### RenderMain

- **Purpose:** Coordinates the 3D-to-2D rendering pipeline: visibility determination via portal-based ray-casting, depth-sorted polygon decomposition, sprite/object placement, and rasterization to framebuffer (software or OpenGL backend).

- **Key directories / files:**
  - `Source_Files/RenderMain/{render.cpp, NewRenderVisTree.cpp, RenderSortPoly.cpp, RenderPlaceObjs.cpp, RenderRasterize.cpp}`
  - `Source_Files/RenderMain/{Rasterizer.h, Rasterizer_SW.h, Rasterizer_OGL.h}`
  - `Source_Files/RenderMain/{scottish_textures.cpp, texturers.cpp, AnimatedTextures.cpp}`
  - `Source_Files/RenderMain/{shapes.cpp, images.h, ImageLoader_*.cpp, DDS.h}`

- **Key responsibilities:**
  - Build portal-based visibility tree; determine visible polygons from player viewpoint via ray-casting
  - Decompose visibility tree into back-to-front depth-sorted polygon list with screen-space clipping regions
  - Place game objects (sprites, monsters, items) in sorted depth order; compute 2D screen projections
  - Clip and rasterize textured polygons to framebuffer; dispatch to Rasterizer_SW (software) or Rasterizer_OGL (OpenGL)
  - Apply texture mapping with DDA coordinate interpolation; support perspective-correct mapping, animated textures, and transfer modes
  - Calculate lighting and effects (shading tables, distance/angle lighting, transparency modes)
  - Load and cache sprite/texture collections; manage color lookup tables, shading tables, and animation state
  - Decompress RLE and DXTC bitmap formats on-the-fly

- **Key dependencies (other subsystems):**
  - **Map/Geometry:** Consumes `polygon_data`, `side_data`, `endpoint_data`, `line_data` for world construction
  - **Objects/Sprites:** Game object positions, animation states from `map.h` structures
  - **Light sources:** Intensity lookups from `lightsource.h` for per-surface shading
  - **Media/Liquids:** Boundary checks from `media.h` for transparency and clipping
  - **View/Camera:** FOV, position, orientation, viewport from `render.h`, `ViewControl.h`
  - **Preferences:** Rendering mode (software vs. OpenGL), graphics quality settings
  - **Weapons display:** Weapon sprite/animation for foreground layer
  - **AnimatedTextures:** Per-frame texture animation updates

---

### RenderOther

- **Purpose:** Manages all 2D visual output: screen mode initialization, per-frame rendering orchestration, HUD/UI rendering, image and font resource loading, and platform-specific graphics API backends (SDL software, OpenGL, macOS Quickdraw).

- **Key directories / files:**
  - `Source_Files/RenderOther/{screen.cpp, screen_drawing.cpp, game_window.cpp}`
  - `Source_Files/RenderOther/{HUDRenderer*.cpp, images.cpp, fades.cpp}`
  - `Source_Files/RenderOther/{FontHandler.cpp, Shape_Blitter.cpp, Image_Blitter.cpp}`
  - `Source_Files/RenderOther/{overhead_map.cpp, computer_interface.cpp, motion_sensor.cpp}`
  - `Source_Files/RenderOther/{ChaseCam.cpp, ViewControl.cpp, screen_definitions.h}`

- **Key responsibilities:**
  - Initialize display modes (resolution, bit depth, windowed/fullscreen); allocate SDL rendering surfaces (`world_pixels`, `HUD_Buffer`)
  - Orchestrate per-frame rendering pipeline: clear buffers ΓåÆ invoke RenderMain::render_view() ΓåÆ draw HUD ΓåÆ render overlays ΓåÆ update screen
  - Implement abstract HUD rendering with concrete backends (SDL software, OpenGL, Lua scripting); update weapon, ammo, shield, oxygen, inventory displays
  - Load and manage 2D image resources (PICT, WAD-embedded) with format conversion, rescaling, surface caching
  - Manage font lifecycle (bitmap fonts, TrueType via SDL_ttf, OpenGL texture-based glyphs); calculate text width
  - Render screen effects (color fades, tinting, gamma correction, tunnel vision, teleport feedback)
  - Render minimap/automap with entity markers, line visibility, polygon shading (SDL and OpenGL backends)
  - Display interactive terminal/computer interface with rich text, input handling
  - Display motion sensor (radar) with entity tracking and team compass
  - Provide 2D drawing primitives (shapes, images, rectangles, lines, text)
  - Manage third-person camera (ChaseCam) with spring-damper physics

- **Key dependencies (other subsystems):**
  - **RenderMain:** Invokes `render_view()` to draw 3D world; consumes view data
  - **Map/World:** Geometry, polygons, lines, endpoints for automap rendering
  - **Player/Objects:** Position, animation for entity rendering
  - **Monsters:** State for motion sensor tracking
  - **Network:** Multiplayer state, player rankings, microphone status
  - **Shape/Image resources:** Shape descriptors, interface resource IDs
  - **XML:** Configuration parsing for interface rectangles, colors, fonts
  - **SDL, OpenGL (conditional):** Graphics API backends
  - **FileHandler:** Resource file I/O (WAD, image files)

---

### Sound

- **Purpose:** Manages all audio playback: music, sound effects, 3D spatial audio positioning, ambient environmental sounds, and network microphone data. Provides unified interface through `SoundManager` singleton while delegating low-level mixing and output to SDL.

- **Key directories / files:**
  - `Source_Files/Sound/{SoundManager.cpp, Mixer.cpp, Music.cpp}`
  - `Source_Files/Sound/{*Decoder.cpp}` (BasicIFFDecoder, MADDecoder, VorbisDecoder, SndfileDecoder)
  - `Source_Files/Sound/{SoundFile.cpp, ReplacementSounds.cpp}`
  - `Source_Files/Sound/{sound_definitions.h, song_definitions.h, SoundManagerEnums.h}`

- **Key responsibilities:**
  - Initialize and shutdown SDL audio subsystem with configurable sample rates, bit depths, channel counts
  - Load and decode audio files in multiple formats (WAV, AIFF, MP3, OGG Vorbis, FLAC) via pluggable decoder abstraction
  - Mix multiple concurrent audio channels in real-time using fixed-point arithmetic for pitch and resampling
  - Calculate 3D spatial audio with distance-based attenuation, stereo panning, obstruction-aware volume falloff
  - Manage sound definitions with permutations and depth-based volume curves
  - Play and stop music with crossfading and level-based playlist sequencing
  - Support external sound replacement via registry with hash-accelerated lookup
  - Update up to 4 concurrent ambient environmental sounds (looping)
  - Allocate and prioritize audio channels using LRU cache with priority-based eviction
  - Apply master volume, per-channel volume, and voice ducking (attenuation during network transmission)

- **Key dependencies (other subsystems):**
  - **FileHandler:** File I/O abstraction (`FileSpecifier`, `OpenedFile`)
  - **Game state:** `local_player_index`, `dynamic_world` for context-aware audio behavior
  - **Network audio:** Microphone data queuing/dequeuing for network voice playback
  - **Random numbers:** `GM_Random::KISS()` for sound selection and playlist shuffling
  - **Game engine callbacks:** `_sound_listener_proc()`, `_sound_obstructed_proc()`, `_sound_add_ambient_sources_proc()` for 3D positioning
  - **SDL, libmad, libvorbisfile, libsndfile (conditional):** Audio decoding and output backends
  - **Logging:** Diagnostic output via `logWarning3()`

---

### TCPMess

- **Purpose:** Provides non-blocking TCP-based message networking infrastructure for synchronizing game state between connected clients. Handles message framing, serialization/deserialization, routing to type-specific handlers, and connection lifecycle.

- **Key directories / files:**
  - `Source_Files/Network/TCPMess/{CommunicationsChannel.cpp, Message.cpp, MessageInflater.cpp, MessageDispatcher.cpp, MessageHandler.cpp}`

- **Key responsibilities:**
  - Establish and manage TCP socket connections (connect/disconnect) with non-blocking mode
  - Perform message framing with magic-number validation and length headers
  - Buffer partial I/O across multiple socket operations using state machine
  - Manage paired inbound/outbound message queues
  - Deserialize wire-format bytes into type-specific `Message` subclasses using prototype factory
  - Route dispatched messages to type-specific handlers via `std::map` lookup
  - Support synchronous message receive with overall and inactivity timeout semantics
  - Track activity timestamps for timeout detection
  - Accept incoming connections via factory pattern (server-side)

- **Key dependencies (other subsystems):**
  - **SDL_net:** Socket creation, send/recv operations, socket set management, peer address lookup
  - **AStream classes:** Big-endian stream serialization/deserialization
  - **SDL:** Timestamp retrieval (`SDL_GetTicks`)
  - **Platform APIs:** Windows winsock2, macOS OpenTransport, Unix BSD sockets

---

### XML

- **Purpose:** Provides configuration file parsing infrastructure using the Expat C XML parser. Implements hierarchical element parser tree with type-safe value reading and platform-specific data adapters (file I/O, memory buffers, macOS resource forks).

- **Key directories / files:**
  - `Source_Files/XML/{XML_ElementParser.cpp, XML_Configure.cpp, XML_DataBlock.cpp}`
  - `Source_Files/XML/{XML_Loader_SDL.cpp, XML_ResourceFork.cpp, XML_ParseTreeRoot.h}`
  - `Source_Files/XML/{XML_LevelScript.cpp, ColorParser.cpp, DamageParser.cpp, ShapesParser.cpp}`
  - `Expat/` (bundled Expat XML parser library)

- **Key responsibilities:**
  - Parse XML configuration files from multiple sources (disk files, in-memory buffers, macOS resource forks)
  - Implement hierarchical element parser tree with parent-child relationships and lifecycle hooks (Start, End, HandleAttribute, HandleString)
  - Provide type-safe value parsing with bounds validation (integers, floats, booleans, UTF-8 conversion)
  - Delegate element-specific parsing to subsystem-specific parsers (colors, damage, shapes)
  - Load and execute level-specific scripts (MML, Lua, music, movies) from map file resources
  - Collect and report three error categories (read, parse, interpretation) with line numbers
  - Support case-insensitive tag/attribute matching and value validation

- **Key dependencies (other subsystems):**
  - **Expat:** XML parser C library
  - **FileHandler:** File I/O abstraction, directory enumeration
  - **Images subsystem:** Load embedded resources
  - **Lua subsystem:** Script execution
  - **Music subsystem:** Level music management
  - **OGL_LoadScreen subsystem:** Load screen display
  - **cseries.h:** Platform abstractions and utilities

---

### tools

- **Purpose:** Development and debugging utilities for inspecting and manipulating Marathon resource and WAD files.

- **Key directories / files:**
  - `tools/{dumprsrcmap.cpp, dumpwad.cpp, MapChunkerMain.cpp}`

- **Key responsibilities:**
  - Parse and enumerate Macintosh resource map files (AppleSingle, MacBinary, raw formats)
  - Inspect Marathon WAD file structure, headers, directory entries, mission/environment/entry-point flags
  - Convert between macOS resource fork and WAD chunk formats (images, color tables, sound, text)
  - Provide command-line and GUI interfaces for file inspection and resource inventory reporting

- **Key dependencies (other subsystems):**
  - **resource_manager.cpp:** Resource file parsing
  - **wad.cpp:** WAD file reading and metadata extraction
  - **FileHandler_SDL.cpp:** Cross-platform file I/O
  - **map.h:** Data structure definitions (flags, enumerations)
  - **Platform APIs:** macOS Carbon/QuickTime (MapChunkerMain)

---

## Key Runtime Flows

### Initialization

1. **XML Configuration Setup**
   - `XML_ParseTreeRoot::SetupParseTree()` constructs global parser tree, attaching subsystem-specific parsers
   - Configuration files are loaded via `XML_Loader_SDL`, populating game settings

2. **Display and Rendering Initialization**
   - `RenderOther::initialize_screen()` allocates SDL surfaces (`world_pixels`, `HUD_Buffer`, `Term_Buffer`)
   - Selects display mode (resolution, bit depth)
   - Initializes images manager (`initialize_images_manager()`)
   - Allocates HUD renderer (software via `HUDRenderer_SW`, OpenGL via `HUDRenderer_OGL`, or Lua via `HUDRenderer_Lua`)

3. **RenderMain Asset Loading**
   - Load sprite/texture collections from disk
   - Build shading tables and color lookup tables (CLUTs)
   - Allocate rasterizer state and temporary buffers
   - OpenGL backend (if enabled): Initialize context, detect extensions, pre-load textures (disabled on Dingoo)

4. **Font and Image Resource Loading**
   - `FontHandler` initializes bitmap/TrueType fonts from resources
   - Image caching infrastructure prepared

5. **Audio Initialization**
   - `SoundManager::Initialize()` opens sound resource files via `FileSpecifier`
   - Initializes `Mixer`, calls `SDL_OpenAudio()` with configured `SDL_AudioSpec`
   - Sets up audio callback handler that invokes `Mixer::FillBuffer()`
   - Opens music decoders (lazily for specific level)

6. **Network Initialization** (if networked)
   - Create `CommunicationsChannel` instances for peer connections
   - Register message handlers via `MessageDispatcher`
   - Set up message prototype registry via `MessageInflater`

---

### Per-frame / Main Loop

1. **Input Processing & Game Logic**
   - Input subsystem processes keyboard/controller input (Dingoo shoulder buttons remapped)
   - Game world updates positions, animations, physics

2. **Rendering Pipeline**
   - `RenderOther::render_screen()` orchestrates frame rendering:
     1. Clear framebuffer and depth state
     2. Call `RenderMain::render_view()` to render 3D world:
        - Build visibility tree via `NewRenderVisTree` (ray-casting from player position)
        - Decompose into depth-sorted polygon list via `RenderSortPoly`
        - Place game objects (sprites, monsters) in depth order via `RenderPlaceObjs`
        - Rasterize via `RenderRasterize` to selected backend (Rasterizer_SW or Rasterizer_OGL)
     3. Draw HUD layer:
        - Invoke `HUDRenderer` to update weapon/ammo/shield/oxygen/inventory displays
        - Check dirty flags to avoid full redraws
     4. Render overlays (crosshairs, motion sensor, terminal, overhead map)
     5. Render foreground weapon/HUD in screen-space
     6. Update animated texture state for next frame

3. **Audio Updates** (asynchronous in SDL audio thread)
   - `Music::Idle()` updates music state (fade transitions, buffer refills, playlist advancement)
   - `SoundManager` maintains active channels and updates ambient sound sources
   - Real-time mixing occurs in `Mixer::FillBuffer()` callback

4. **Network Message Processing**
   - `CommunicationsChannel::PumpInbound()` reads TCP data, frames messages
   - `MessageInflater` deserializes framed bytes into typed `Message` objects
   - `MessageDispatcher` routes messages to registered handlers by type ID
   - Outbound messages queued via `CommunicationsChannel::sendMessage()` are serialized and flushed to TCP

5. **Screen Update**
   - SDL surface blit: copy rendered framebuffer to video memory
   - SDL_Flip() or OpenGL buffer swap

---

### Shutdown

1. **Audio Shutdown**
   - `SoundManager::Shutdown()` stops all active sounds
   - Close music decoders
   - Call `SDL_CloseAudio()` to shut down SDL audio subsystem

2. **RenderMain Cleanup**
   - Unload sprite/texture collections
   - Deallocate shading tables and CLUTs
   - Free rasterizer state and temporary buffers
   - OpenGL backend (if enabled): Delete textures, unload 3D models, destroy context

3. **RenderOther Cleanup**
   - Deallocate HUD renderer and surfaces (`world_pixels`, `HUD_Buffer`)
   - Unload fonts, image resources
   - Cleanup OpenGL textures and display lists (if enabled)

4. **XML Cleanup**
   - Close file handles for configuration files
   - Deallocate parser tree

5. **Network Shutdown**
   - Close all `CommunicationsChannel` socket connections
   - Deallocate message handler registry

---

## Data & Control Boundaries

- **Global State Ownership:**
  - `local_player_index`: Read-only reference to current player; shared across subsystems
  - `dynamic_world`: Read-only reference to game world state (polygons, lines, objects); shared across subsystems
  - `RenderMain` and `RenderOther` coordinate view/viewport parameters via `view_data` structures

- **Memory Ownership:**
  - `RenderOther` owns SDL rendering surfaces: `world_pixels` (3D framebuffer), `HUD_Buffer` (HUD layer), `Term_Buffer` (terminal display)
  - `SoundManager` owns cached sound data with LRU eviction; when capacity exceeded, least-recently-used sounds are unloaded
  - `RenderMain` owns collection/shading/CLUT/animation state during rendering
  - XML parser owns configuration state; can be reset via `ResetAllMMLValues()`

- **Audio Threading Boundary:**
  - `Mixer::FillBuffer()` runs in SDL audio callback thread
  - Critical sections protected by `SDL_LockAudio()` / `SDL_UnlockAudio()` to serialize access to channel state
  - `SoundManager` and `Music` manage mutable state (channel list, music decoder, animation frame counter) with thread-safe updates

- **Message Dispatch Boundary:**
  - `MessageDispatcher` maintains type-to-handler mapping (`std::map<MessageTypeID, MessageHandler*>`)
  - Handlers are registered at initialization and invoked from receive thread context
  - Handler callables can be function pointers or member methods via template adapters

- **Configuration Lifetime:**
  - XML configuration loaded once at startup via `SetupParseTree()` and file parsing
  - Level-specific scripts loaded per-level via `XML_LevelScript::Load()` from map resource 128
  - `ResetAllMMLValues()` resets to hardcoded defaults without reload

- **Resource Resolution:**
  - `FileSpecifier` abstraction hides platform-specific file I/O (SDL on cross-platform, macOS resource forks conditionally)
  - Images, fonts, and sounds resolved via `FileHandler` callback subsystem
  - Network messages use prototype factory pattern; no hardcoded deserialization

---

## Notable Risks / Hotspots

- **Dingoo Platform Constraints:**
  - 400 MHz Ingenic JZ4740 MIPS32 SoC: Portal visibility tree construction and polygon rasterization are CPU-critical paths
  - 32 MB RAM: Sound cache LRU eviction and RLE/DXTC decompression add CPU overhead; texture memory footprint tightly constrained
  - 320├ù240 fixed resolution: Hardcoded in `screen_shared.h` and rendering constants; no runtime scaling
  - Software-only rendering: OpenGL entirely excluded (`#ifndef HAVE_OPENGL`); no hardware acceleration available

- **Binary Size Optimization:**
  - `SW_Texture_Extras.h` conditionally disabled (`#ifdef HAVE_DINGOO`) to reduce opacity lookup tables
  - texturers.cpp template instantiation generates separate functions for each color depth; conditional compilation avoids all unnecessary instantiations

- **Visibility Tree Performance:**
  - `NewRenderVisTree` uses portal-based ray-casting with `long_point2d` extended-precision vectors to prevent overflow in large worlds
  - Incorrect visibility culling leads to rendering invisible surfaces (CPU waste) or missing visible surfaces (visual errors)

- **Audio Callback Threading:**
  - Concurrent access to `SoundManager` channel list and `Music` decoder state from main thread and audio callback thread
  - Lock contention on `SDL_LockAudio()` / `SDL_UnlockAudio()` during per-frame audio updates and real-time mixing

- **Network Message Deserialization:**
  - `MessageInflater` prototype factory relies on correct type ID registration; unregistered message types silently dropped
  - Fragmented TCP frames may require multiple pump cycles to complete message frame

- **Polygon Clipping State Machine:**
  - `RenderRasterize` uses procedural clipping across XY, Z, and XZ coordinate spaces; complex state transitions prone to edge-case bugs
  - Viewport frustum and portal boundary clipping interactions must be validated

- **Animated Texture Update Latency:**
  - Animated texture frame advancement happens once per render frame; frame-rate stuttering causes animation jank
  - Sprite animation permutation selection is probabilistic; synchronization with game logic frame may diverge

- **Resource Loading and Caching:**
  - Image rescaling and RLE/DXTC decompression occur on-demand during rendering; blocking I/O in render pipeline can cause frame drops
  - Font glyph texture generation (OpenGL) may stall render thread during first text render of new glyph set

- **Fixed-Point Resampling Precision:**
  - `Mixer` uses fixed-point arithmetic (`_fixed` type) for pitch and resampling; accumulated rounding errors in long-playing sounds
  - Linear interpolation in texture mapping may introduce aliasing artifacts at extreme magnification ratios
