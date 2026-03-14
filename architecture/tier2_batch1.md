# Architecture Overview

## Repository Shape

The codebase mirrors a cross-platform game engine with platform-specific variants gated behind preprocessor conditionals. The main structure is:

- **Source_Files/** ΓÇö Core game engine subsystems (mirrors mainline Aleph One layout):
  - CSeries/ ΓÇö Cross-platform foundational utilities
  - Files/ ΓÇö File I/O and asset management
  - GameWorld/ ΓÇö Game simulation (player, entities, physics, world state)
  - Input/ ΓÇö Hardware input abstraction
  - Misc/ ΓÇö Infrastructure (logging, preferences, state machine, UI framework, action queues)
  - Network/ ΓÇö Multiplayer networking subsystems
  - RenderMain/, RenderOther/ ΓÇö Rendering pipeline (not detailed in this batch; see next batch)
  - Sound/ ΓÇö Audio subsystem (likely reduced for Dingoo; see next batch)
  - XML/ ΓÇö XML parsing and game config integration
  - Scripting/ ΓÇö Lua VM and bindings (optional, gated by `HAVE_LUA`)
  - ModelView/ ΓÇö 3D model loaders and skeletal animation
- **dingux-build/** ΓÇö MIPS cross-compile Makefile and build-specific configuration
- **Expat/** ΓÇö Bundled Expat XML parser (independent subsystem)
- **Extras/** ΓÇö Build-time utilities (off-line tools: resource extraction, WAD generation)
- **Extra OpenGL Docs/** ΓÇö Reference materials for OpenGL techniques (not integrated into runtime)
- **PBProjects/** ΓÇö macOS build configuration and feature flags (Dingoo uses dingux-build instead)
- **Prefix/** ΓÇö Platform-specific prefix headers (Dingoo uses autoconf config.h)

---

## Major Subsystems

### CSeries
- **Purpose:** Foundational cross-platform compatibility layer and core utilities providing type abstractions, platform detection, string handling, dialogs, graphics management, binary serialization, timing, and error handling.
- **Key directories / files:**
  - `cseries.h` ΓÇö Master include aggregating type definitions, platform detection, and utilities
  - `cstypes.h` ΓÇö Fixed-size integer types, fixed-point arithmetic (16.16), platform constants
  - `csstrings.h/.cpp` ΓÇö Pascal/C string conversion, encoding (Mac Roman Γåö Unicode Γåö UTF-8)
  - `csalerts.h/.cpp`, `csdialogs.h/.cpp` ΓÇö Dialog/alert management (Mac Carbon native vs. SDL emulation)
  - `BStream.h/.cpp`, `byte_swapping.h/.cpp` ΓÇö Binary serialization with explicit big-endian/little-endian control
  - `timer.h/.cpp`, `mytm.h/.cpp` ΓÇö Timing (tick counter, periodic task callbacks)
  - `gdspec.h/.cpp`, `cspixels.h` ΓÇö Graphics device enumeration, pixel format conversion macros
  - `csfiles.h/.cpp` ΓÇö File specification queries (FSSpec abstraction)
  - `my32bqd.h/.cpp` ΓÇö QuickDraw GWorld (offscreen rendering) abstraction
  - `readout_data.h` ΓÇö Global performance profiling counters
- **Key responsibilities:**
  - Type definitions and fixed-point arithmetic for platform-neutral code
  - Platform detection and conditional subsystem initialization (SDL vs. Carbon, endianness detection)
  - String encoding converters for cross-platform resource/file compatibility
  - Binary stream serialization with automatic endianness conversion (all game data in big-endian)
  - Dialog creation and control manipulation abstraction (two parallel implementations: Mac Carbon and SDL emulation)
  - System timing (millisecond ticker, periodic task callbacks)
  - Graphics device enumeration and pixel format conversions
  - Performance instrumentation (frame timing, rendering statistics)
- **Key dependencies:** SDL library, Mac Carbon/Cocoa (platform-specific), standard C/C++ libraries
- **Dingoo-specific notes:** SDL-based implementations (_sdl.cpp variants) replace Carbon calls; fixed 320x240 resolution assumed in dialog layout; timer and graphics device queries simplified for embedded Linux.

---

### Files
- **Purpose:** Comprehensive file I/O abstraction managing game data persistence, asset discovery, WAD container format, resource file handling (macOS resource forks, AppleSingle, MacBinary), and binary serialization with cross-platform endianness control.
- **Key directories / files:**
  - `AStream.h/.cpp` ΓÇö Templated binary stream with explicit endianness control and bounds checking
  - `Packing.h/.cpp` ΓÇö Byte-stream serialization utilities for 16/32-bit integers
  - `FileHandler.h/.cpp`, `FileHandler_SDL.cpp` ΓÇö Cross-platform file abstraction (FSSpec ΓåÆ SDL_RWops)
  - `resource_manager.h/.cpp` ΓÇö MacOS resource file parser (native, AppleSingle, MacBinary II/III)
  - `wad.h/.cpp`, `game_wad.h/.cpp` ΓÇö WAD file format (Where's All the Data) for asset containers and game state
  - `tags.h` ΓÇö Typecode enum and four-character tags for WAD chunks
  - `crc.h/.cpp` ΓÇö CRC32/CRC16 checksums for data integrity
  - `find_files.h/.cpp`, `find_files_sdl.cpp` ΓÇö Recursive asset discovery with type-based filtering
  - `preprocess_map_*.cpp` ΓÇö Asset search, save/load dialogs, automatic game saves with thumbnails
  - `wad_prefs.h/.cpp` ΓÇö Preferences persistence using WAD format
  - `extensions.h` ΓÇö Physics file management
- **Key responsibilities:**
  - Abstract platform-specific file operations (macOS FSSpec, Windows API, POSIX) via FileSpecifier and DirectorySpecifier
  - Implement binary serialization with explicit big-endian byte order (Marathon standard)
  - Parse and manage MacOS resource files with multi-file stacking
  - Define and implement WAD container format for tagged game assets with backwards-compatible versioning
  - Discover and load game assets (maps, physics, shapes, sounds) from configurable search paths
  - Persist game state (player position, inventory, level progress) to disk
  - Validate data integrity via CRC32 checksums
  - Map abstract file type codes (Typecode enum) to platform-specific file type identifiers
- **Key dependencies:** CSeries, GameWorld (for map/entity serialization), Network (for map distribution)
- **Dingoo-specific notes:** SDL-based implementation (FileHandler_SDL.cpp, find_files_sdl.cpp); game state isolated per episode via `ALEPHONE_USERDATA` environment variable to manage 32 MB RAM constraints; no macOS resource fork support on target platform.

---

### GameWorld
- **Purpose:** Complete game world simulation for constrained MIPS hardware: polygon-based geometry, interactive entities (player, monsters, items, projectiles, effects, scenery), physics simulation, environmental systems (platforms, lights, liquids), and main game loop orchestrating per-frame updates.
- **Key directories / files:**
  - `map.h/.cpp`, `map_constructors.cpp` ΓÇö World geometry (polygons, lines, endpoints, sides), collision detection, spatial indices, entity lifecycle
  - `world.h/.cpp` ΓÇö Coordinate system, fixed-point arithmetic, trigonometric lookups, deterministic random number generation
  - `marathon2.cpp` ΓÇö Main game loop: initialization, per-tick updates, level transitions, action queue processing, completion checks
  - `player.h/.cpp` ΓÇö Player entity state, inventory, damage/healing, oxygen mechanics, death/respawn, network synchronization
  - `physics.h/.cpp`, `physics_models.h` ΓÇö Player physics (position, velocity, gravity, jumping, collision, media interaction)
  - `monsters.h/.cpp`, `monster_definitions.h` ΓÇö NPC lifecycle, frame-by-frame AI, pathfinding, combat behavior
  - `flood_map.h/.cpp`, `pathfinding.cpp` ΓÇö Polygon-adjacency pathfinding with breadth-first/best-first traversal and custom cost evaluation
  - `projectiles.h/.cpp`, `projectile_definitions.h` ΓÇö Projectile physics, collision, damage-on-impact, detonation effects
  - `effects.h/.cpp`, `effect_definitions.h` ΓÇö Temporary visual/audio phenomena (~60+ effect types)
  - `platforms.h/.cpp`, `platform_definitions.h` ΓÇö Movable platforms (doors, elevators, crushers) with state machines and geometry updates
  - `lightsource.h/.cpp` ΓÇö Dynamic lighting with 6-state machine and intensity animation
  - `media.h/.cpp`, `media_definitions.h` ΓÇö Liquid media (water, lava, goo, sewage) with damage and detonation effects
  - `devices.cpp` ΓÇö Interactive control panels (recharging, toggles, save points)
  - `items.h/.cpp`, `item_definitions.h` ΓÇö Item placement and pickup mechanics
  - `scenery.h/.cpp`, `scenery_definitions.h` ΓÇö Static/destroyable world objects with animation and randomization
  - `weapons.h/.cpp`, `weapon_definitions.h` ΓÇö Weapon state machines (idle, firing, reloading), dual-trigger mechanics
  - `dynamic_limits.h/.cpp` ΓÇö Runtime-configurable entity count limits (XML-driven with fallback defaults)
  - `TickBasedCircularQueue.h` ΓÇö Tick-indexed circular queue for per-frame action flags
- **Key responsibilities:**
  - Maintain polygon-based world geometry with collision and spatial query support
  - Create, update, and destroy all interactive entities within determinism and network-sync constraints
  - Simulate player physics (movement, gravity, jumping, media interaction, collision)
  - Implement monster AI (pathfinding, target acquisition, combat, animation)
  - Calculate projectile ballistics and collision damage
  - Manage combat system (damage application, powerups, invincibility, death/respawn)
  - Orchestrate environmental interactions (movable platforms, dynamic lighting, liquid physics)
  - Process inventory and item pickup mechanics
  - Handle device interaction (terminals, recharging, toggles)
  - Execute main game loop with per-frame updates across all subsystems
  - Serialize/deserialize complete world state for save/load and network transmission
  - Apply XML-driven customization for all game entities and mechanics
- **Key dependencies:** CSeries (types, timing, math), Files (map loading, save/load), Misc (action queues, preferences, logging), Network (player sync, multiplayer state), Input (action flags), Lua (optional scripting hooks)
- **Dingoo-specific notes:** All coordinates and physics use fixed-point arithmetic (16.16) for deterministic simulation on MIPS. Dynamic entity limits proportional to available 32 MB RAM. Pathfinding cost-evaluation can be customized to avoid expensive calculations. Lua scripting likely disabled (`#undef HAVE_LUA`). Deterministic RNG essential for multiplayer sync on low-power hardware.

---

### Input
- **Purpose:** Abstracts hardware input devices (keyboards, mice, joysticks) and converts device state into normalized game action flags and movement deltas (yaw, pitch, velocity) with user preference scaling.
- **Key directories / files:**
  - `joystick_sdl.cpp` ΓÇö SDL joystick polling, button-to-keypress mapping, analog axis scaling with dead zones and pulse-modulation
  - `joystick.h` ΓÇö Public interface for joystick lifecycle and keymap conversion
  - `mouse_sdl.cpp` ΓÇö SDL mouse movement capture, delta normalization, scroll wheel-to-weapon cycling, button simulation
  - `mouse.h` ΓÇö Public interface for mouse lifecycle and per-frame polling
  - `ISp_Support.cpp/.h` ΓÇö macOS InputSprocket framework integration (44 action definitions, device polling)
- **Key responsibilities:**
  - Poll joystick buttons each frame and translate to virtual keycodes via user keymap
  - Scale joystick analog axes (strafe, velocity, yaw, pitch) via sensitivity/dead zones
  - Implement pulse-modulation for strafing (discrete intensity bands instead of smooth analog)
  - Capture and normalize mouse movement deltas with sensitivity and acceleration curves
  - Map mouse buttons and scroll wheel to game actions
  - Apply user preference settings (sensitivity, axis inversion, acceleration, button bindings) to all input
  - Maintain thread-safe mouse state on platforms using Carbon Event Manager
- **Key dependencies:** CSeries (types, timing), Misc (preferences, action flag definitions)
- **Dingoo-specific notes:** Joystick/gamepad input primary (no mouse on handheld). Shoulder button remapping gated behind `#ifdef HAVE_DINGOO`. Pulse-modulation essential for constrained input (no analog sticks on A320). Context-sensitive input handling for map view vs. game view.

---

### Misc
- **Purpose:** Cross-cutting infrastructure including error handling, hierarchical logging, user preferences persistence and UI, game state machine, input recording/replay, console scripting, dialog/widget frameworks, and platform-specific implementations for networking, threading, and UI.
- **Key directories / files:**
  - `interface.h/.cpp` ΓÇö Central game state machine managing lifecycle transitions (intro ΓåÆ menu ΓåÆ gameplay ΓåÆ epilogue ΓåÆ quit), screen rendering, network coordination
  - `vbl.h/.cpp` ΓÇö Input polling, action flag generation, game recording/replay with run-length compression
  - `preferences.h/.cpp` ΓÇö User preferences (graphics, sound, input, network, player) with XML persistence and dialog management
  - `Logging.h/.cpp` ΓÇö Hierarchical logging with configurable severity levels, context stacks, file routing, XML configuration
  - `Console.h/.cpp` ΓÇö Interactive console with command parsing, macro expansion, kill message reporting
  - `sdl_dialogs.h/.cpp`, `sdl_widgets.h/.cpp` ΓÇö SDL-based modal dialog framework with theme loading and widget library
  - `ActionQueues.h/.cpp` ΓÇö Multi-player action queue management with circular buffers and zombie player handling
  - `game_errors.h/.cpp` ΓÇö Error state tracking
  - `PlayerImage_sdl.h/.cpp` ΓÇö Player sprite rendering with lazy-loading of shape data
  - `key_definitions.h` ΓÇö Keyboard configurations with three preset layouts and Dingoo shoulder button support
  - `DefaultStringSets.cpp` ΓÇö Compiled-in UI strings (replacing macOS resource files)
  - `LocalEvents.h` ΓÇö Bitflag-based local event system (quit, pause, zoom, debug)
  - `Scenario.h/.cpp` ΓÇö Scenario metadata and version compatibility
  - `thread_priority_sdl*.cpp` ΓÇö Platform-specific thread priority boosting
  - `sdl_network.h` ΓÇö SDL-based networking abstraction (DDP/UDP, ADSP/TCP)
  - `VecOps.h` ΓÇö Template-based 3D vector operations
  - Additional: `Random.h` (Marsaglia RNG), `CircularQueue.h`, `CircularByteBuffer.h`, `binders.h` (state sync), `progress.h` (loading UI), `OGL_Dialog.cpp`, `alephversion.h`
- **Key responsibilities:**
  - Implement game state machine (init, menu, gameplay, epilogue, shutdown)
  - Poll hardware input each frame and generate action flag bitsets
  - Record/replay game sessions to disk with compression and map/player metadata
  - Persist and load user preferences (graphics, sound, input, network, player) from XML
  - Route log messages by severity level with hierarchical context and domain-based filtering
  - Execute console commands with macro expansion and kill message customization
  - Provide cross-platform SDL and macOS Carbon widget libraries (buttons, text entry, lists, tabs, color pickers)
  - Manage multi-player action queues with circular buffers and zombie player restrictions
  - Track error state (type and code) for recovery and diagnostics
  - Support bidirectional state synchronization between UI controls and data via Bindable/Binder pattern
  - Provide thread priority boosting (Windows, POSIX, macOS)
  - Dispatch local events (quit, pause, sound volume, map zoom, debug)
  - Manage scenario metadata and version compatibility
  - Support three preset keyboard layouts with custom XML-based overrides
- **Key dependencies:** CSeries (types, dialogs), GameWorld (state queries), Files (I/O), Network (game mode state), Audio (Music, SoundManager), XML (parsing preferences/logging config)
- **Dingoo-specific notes:** Fixed 320x240 resolution baked into UI layout. Keyboard mappings include Dingoo shoulder buttons. Thread priority boosting is stub/fallback only. Networking infrastructure may be reduced or disabled (`#ifdef HAVE_DINGOO`). Preferences stored in SDL-compatible format, not Mac resources. Input heartbeat polled at ~30 Hz.

---

### Network
- **Purpose:** Enables multiplayer networking with player gathering/joining, game-data distribution, action flag synchronization, optional voice communication, service discovery (SSLP LAN, metaserver internet), and game-mode-specific logic (KoTH, CTF, etc.).
- **Key directories / files:**
  - `network.h/.cpp` ΓÇö Core API and state machine for multiplayer topology initialization and player lifecycle
  - `network_messages.h/.cpp` ΓÇö TCPMess protocol (HelloMessage, TopologyMessage, ActionMessage, etc.)
  - `network_private.h` ΓÇö Internal structures (packet headers, ring/star protocol formats, error codes)
  - `RingGameProtocol.h/.cpp` ΓÇö Ring-topology implementation (daisy-chain forwarding, exponential-backoff retransmission)
  - `StarGameProtocol.h/.cpp`, `network_star.h`, `network_star_hub.cpp`, `network_star_spoke.cpp` ΓÇö Star-topology implementation (hub aggregates, spokes relay)
  - `network_games.h/.cpp` ΓÇö Game-mode rules (King of Hill, Capture Flag, etc.), ranking, end conditions
  - `network_data_formats.h/.cpp` ΓÇö Cross-platform wire-format structures with `netcpy()` endianness conversion
  - `network_capabilities.h/.cpp` ΓÇö Capability versioning for protocol negotiation (gameworld, Lua, Speex, compression)
  - `ConnectPool.h/.cpp` ΓÇö Non-blocking TCP connection pooling
  - `CommunicationsChannel.h/.cpp` ΓÇö TCP message buffering and delivery
  - `network_lookup_sdl.h/.cpp` ΓÇö SSLP-based LAN service discovery
  - `SSLP_*.h/.cpp` ΓÇö Simple Service Location Protocol (UDP broadcast service advertisement)
  - `network_dialogs.h/.cpp` ΓÇö Abstract dialog layer for gather/join UI
  - `metaserver/network_metaserver.h/.cpp` ΓÇö Metaserver client (login, game listing, chat)
  - `network_microphone_shared.h/.cpp`, `network_speaker_*.cpp`, `network_speex.h/.cpp` ΓÇö Voice communication (audio capture/playback, Speex codec)
  - `SDL_netx.h/.cpp` ΓÇö SDL_net extension for UDP broadcast
  - `network_udp.cpp` ΓÇö Cross-platform UDP via SDL_net with dedicated reception thread
  - `network_dummy.cpp` ΓÇö Stub implementations when networking disabled
  - `Update.h/.cpp` ΓÇö Background thread for online update checking
- **Key responsibilities:**
  - Initialize and maintain ring or star network topologies with player lifecycle (gather/join)
  - Route incoming TCP/UDP packets to protocol handlers
  - Synchronize player action flags across network each game tick with prediction for unconfirmed inputs
  - Distribute game data (maps, physics, Lua scripts, optionally compressed) to joining players
  - Measure latency and adjust playback delay to tolerate network jitter
  - Advertise/locate servers via SSLP (LAN) and metaserver (internet)
  - Route and relay player/team chat messages
  - Capture microphone input, compress with Speex, and stream over network; decode and playback received audio
  - Convert all network structures between host endianness and wire format
  - Pool non-blocking TCP connections for gatherer/joiner negotiation
  - Negotiate protocol capabilities (Lua, Speex, compression support)
  - Track per-player action flag arrival and apply timeout-based drop detection
- **Key dependencies:** CSeries (types, sockets via SDL), Files (map/physics distribution via WAD), GameWorld (action flag delivery, player state), Misc (preferences, action queues, logging, dialogs)
- **Dingoo-specific notes:** Voice communication (Speex, audio capture/playback) likely disabled due to 400 MHz processor and 32 MB RAM constraints. Core TCP/UDP networking remains. Ring and star topologies reduce per-player bandwidth vs. hub-and-spoke. Metaserver support may be disabled or reduced. Compression of game data essential for slow 32 MB constraint. No mmap support; all network buffering via malloc.

---

### Lua
- **Purpose:** Embedded scripting interface enabling level designers and modders to customize gameplay behavior via Lua scripts with bindings to game state (players, monsters, objects, map geometry) and event dispatch for game callbacks.
- **Key directories / files:**
  - `lua_script.h/.cpp` ΓÇö Main script event dispatcher, lifecycle management, state serialization
  - `lua_hud_script.h/.cpp` ΓÇö HUD script loader and lifecycle
  - `lua_templates.h` ΓÇö C++ template metaprogramming for C++/Lua object bindings
  - `lua_hud_objects.h/.cpp` ΓÇö HUD bindings (screen, player, game state, renderer)
  - `lua_map.h/.cpp` ΓÇö Map geometry bindings (polygons, lines, sides, lights, platforms)
  - `lua_monsters.h/.cpp` ΓÇö Monster object and type bindings
  - `lua_objects.h/.cpp` ΓÇö World object bindings (effects, items, scenery, sounds)
  - `lua_player.h/.cpp` ΓÇö Player state bindings (position, inventory, weapons, camera)
  - `lua_projectiles.h/.cpp` ΓÇö Projectile object and type bindings
  - `lua_serialize.h/.cpp` ΓÇö Binary serialization/deserialization of Lua values
  - `lua*.h` (lapi, lauxlib, lcode, etc.) ΓÇö Lua 5.1 VM core (execution, memory, garbage collection)
  - `lua_mnemonics.h`, `language_definition.h` ΓÇö String-to-integer mappings for game constants
- **Key responsibilities:**
  - Load Lua scripts and manage VM lifecycle
  - Dispatch game events to Lua callback functions (player killed, item picked, platform switched, damage taken)
  - Register C++ objects and functions as Lua methods via template-based bindings
  - Expose game entity properties (position, type, health, state) as Lua-accessible fields with getter/setter methods
  - Enable object creation and manipulation from Lua (spawn effects, items, monsters; modify health/position)
  - Provide string mnemonics for game constants (item names, monster types, damage sources)
  - Serialize/deserialize Lua table state for save/load and level transitions
  - Maintain per-script global state (compass override, weapon selection, scoring mode)
  - Support conditional I/O access (solo vs. multiplayer script modes)
- **Key dependencies:** GameWorld (all entity and state bindings), Misc (scripting state, preferences)
- **Dingoo-specific notes:** Entire subsystem gated behind `#ifdef HAVE_LUA` and likely disabled for Dingoo builds due to 400 MHz processor and 32 MB RAM constraints. Lua VM footprint significant for embedded platform. Scripting hooks in GameWorld (lights, platforms, monsters, damage) wrapped in `#ifdef HAVE_LUA`.

---

### ModelView
- **Purpose:** Loads multiple 3D model formats (Wavefront OBJ, QuickDraw 3D/Quesa, 3D Studio Max binary, Dim3 XML) into a unified Model3D structure with skeletal animation support and renders with multi-pass shading and depth-sorting.
- **Key directories / files:**
  - `Model3D.h/.cpp` ΓÇö Core 3D asset container with vertex data, skeletal bones, animation frames/sequences, transformation matrices, bounding box management
  - `ModelRenderer.h/.cpp` ΓÇö Multi-pass renderer with optional Z-buffer, shader callbacks for texturing and per-vertex lighting, depth-sorting
  - `Dim3_Loader.h/.cpp` ΓÇö Dim3 XML model format loader with multi-pass control
  - `WavefrontLoader.h/.cpp` ΓÇö Wavefront OBJ format loader with vertex deduplication and polygon triangulation
  - `QD3D_Loader.h/.cpp` ΓÇö QuickDraw 3D / Quesa format loader with tesselation
  - `StudioLoader.h/.cpp` ΓÇö 3D Studio Max .3DS binary format loader
- **Key responsibilities:**
  - Parse and validate multiple 3D model file formats
  - Populate unified Model3D structure with vertex positions, texture coordinates, normals, vertex colors
  - Manage skeletal bone hierarchy with parent-child relationships
  - Support animation frames (sets of bone poses) and animation sequences (ordered frame lists)
  - Compute vertex positions at specified animation frames with interpolation support
  - Calculate, normalize, and optionally reverse per-vertex normals with smooth edge splitting
  - Render with multi-pass shading via user-supplied shader callbacks
  - Implement depth-sorted rendering (by triangle centroid) when Z-buffer unavailable
- **Key dependencies:** CSeries (types, math, file I/O), GameWorld (coordinate system, trigonometric lookups), Files (FileSpecifier)
- **Dingoo-specific notes:** Software rendering path only (no OpenGL); depth-sorting required for correct blending on framebuffer. 3D models likely minimal or absent on Dingoo due to rendering constraints. All loaders must operate within 32 MB RAM budget.

---

### Expat
- **Purpose:** Bundled callback-driven XML parser library for tokenizing and validating XML documents with support for multiple character encodings, incremental/streaming parsing, and external entity resolution.
- **Key directories / files:**
  - `Expat/xmlparse/xmlparse.cpp/.h` ΓÇö Core parser state machine, callback dispatch, DTD/entity/namespace management
  - `Expat/xmltok/xmltok.cpp/.h` ΓÇö Encoding detection, UTF-8/UTF-16 conversion, tokenizer instantiation
  - `Expat/xmltok/xmltok_impl.cpp/.h` ΓÇö Core XML tokenization (tags, attributes, references, CDATA, comments)
  - `Expat/xmltok/xmlrole.cpp/.h` ΓÇö Prolog state machine, token classification into semantic roles
  - `Expat/xmlparse/hashtable.cpp/.h` ΓÇö Hash table for entity/namespace resolution
  - Character classification tables: `asciitab.h`, `utf8tab.h`, `latin1tab.h`, `nametab.h`
  - `Expat/xmlwf/xmlwf.cpp` ΓÇö XML well-formedness checker (CLI utility)
  - `Expat/xmlwf/xmlfile.cpp` ΓÇö Memory-mapped and stream-based file processing
- **Key responsibilities:**
  - Detect and manage character encodings (UTF-8, UTF-16 LE/BE, ASCII, Latin-1, Windows codepages)
  - Tokenize XML markup (tags, attributes, CDATA, comments, processing instructions, references, declarations)
  - Validate XML grammar via prolog state machine (DOCTYPE, ENTITY, NOTATION, ATTLIST, ELEMENT)
  - Maintain parser state across prolog, content, CDATA, and epilog sections
  - Provide O(1) character classification via pre-computed lookup tables
  - Register and dispatch user-defined callback handlers for parse events
  - Track parse location (line/column) for error reporting
  - Support external entity resolution via child parsers
- **Key dependencies:** Standard C library (memory allocation, stdio)
- **Dingoo-specific notes:** Bundled with Dingoo build. Streaming/incremental design suitable for constrained RAM (no intermediate tree building). Callback-only model minimizes memory allocation. Table-driven character classification optimized for low-power processors.

---

### LibNAT
- **Purpose:** UPnP library for discovering Internet Gateway Devices (IGDs) and managing port mappings to enable NAT traversal for peer-to-peer networking behind firewalls.
- **Key directories / files:**
  - `libnat.h` ΓÇö Public API (IGD discovery, port mapping CRUD, public IP queries, error codes)
  - `error.h`, `http.h`, `ssdp.h` ΓÇö Protocol support layers (error reporting, HTTP GET/POST, SSDP discovery)
  - `os.h`, `os_common.h` ΓÇö Platform abstraction (Unix/Windows socket operations)
  - `utility.h` ΓÇö Macro constants and string utilities
- **Key responsibilities:**
  - Discover UPnP-enabled IGDs on local network via SSDP
  - Establish and remove TCP/UDP port mappings on IGDs
  - Query public IP address from IGDs
  - Provide platform-agnostic socket and HTTP abstractions
- **Key dependencies:** OS socket layer (Unix fcntl/select, Windows winsock), HTTP client, SSDP protocol, XML parser (for device description parsing)
- **Dingoo-specific notes:** Optional library; integration into Network subsystem may be disabled (`#ifdef HAVE_DINGOO`). Primarily useful for internet play via metaserver; reduced priority on embedded platform.

---

## Key Runtime Flows

### Initialization

1. **Engine startup** (shell/main entry point):
   - Load platform config (Prefix headers, autoconf config.h for Dingoo)
   - Initialize CSeries subsystem (types, platform detection, timer, graphics device)
   - Initialize Misc subsystem (logging, preferences from XML/WAD, error state)
   - Load user preferences (graphics, sound, input, network, player settings)
   - Initialize Input subsystem (keyboard layouts from Dingoo key_definitions.h, joystick enumeration)
   - Initialize rendering pipeline (GraphicsState, SDL video mode setup for 320x240)
   - Initialize audio subsystem (likely disabled/stubbed for Dingoo)

2. **Game session start** (main menu ΓåÆ gameplay):
   - Call `NetEnter()` if multiplayer; otherwise set single-player mode
   - Load scenario metadata and validate version compatibility
   - Call `Files::preprocess_map_*()` to search for default assets (maps, physics, shapes, sounds)
   - Open and parse map WAD file (Files subsystem)
   - Construct world geometry (GameWorld::map_constructors)
   - Initialize GameWorld systems: player entity, monster/item/projectile/effect pools
   - Load and unpack physics definitions (Files::import_definitions_from_file)
   - Preload shape collections and sound resources
   - Initialize lights, platforms, media with per-map state
   - If multiplayer, complete network topology handshake and distribute game data (maps, physics, Lua scripts)

3. **Per-frame preparation**:
   - Input subsystem polls hardware (joystick buttons/axes, mouse) and generates normalized action flags
   - Action flags queued locally or transmitted to network (if multiplayer)
   - Lua script `init` callback invoked (if Lua enabled)

### Per-Frame / Main Loop

Executed in `marathon2.cpp` main loop:

1. **Input processing**:
   - Dequeue action flags from local input, network, or replay buffer
   - Apply player preference settings (sensitivity, axis inversion, button remappings)
   - Encode movement deltas (yaw, pitch, velocity) into action flag bitset

2. **Player update** (physics.cpp):
   - Apply action flags to player position, velocity, rotation
   - Resolve collision with terrain (line-of-sight, media boundaries)
   - Update media submersion state (oxygen depletion if in liquid)
   - Apply external forces (gravity, platform riding, monster knockback)

3. **Monster updates** (monsters.cpp):
   - For each monster: read AI state machine phase
   - Pathfinding via flood_map.cpp (polygon adjacency graph, cost-based search)
   - Update position, collision, animation frame
   - Attack detection and damage application
   - Damage feedback (knockback, animation interruption)

4. **Projectile updates** (projectiles.cpp):
   - For each projectile: update position (gravity, wander, guided steering)
   - Collision detection (terrain, monsters, items)
   - Damage application on impact, detonation, or timeout
   - Generate effects (contrails, explosions)

5. **Effect updates** (effects.cpp):
   - For each effect: advance animation frame
   - Play associated sound effects
   - Terminate when animation complete

6. **Platform updates** (platforms.cpp):
   - For each platform: read state machine phase (resting, extending, retracting, crushing)
   - Update polygon heights and endpoint positions
   - Detect and apply player/monster obstruction
   - Trigger associated sounds and effects

7. **Light updates** (lightsource.cpp):
   - For each light: advance state machine phase (becoming active/inactive)
   - Update intensity via animation function (constant, linear, smooth, flicker)
   - Propagate light state to associated media and effects

8. **Media updates** (media.cpp):
   - Update liquid heights from light intensity
   - Apply media-specific damage (lava, goo) to entities
   - Trigger detonation effects

9. **Device updates** (devices.cpp):
   - For each device: update recharge timers (shields, oxygen)
   - Detect player interaction (panel touch)
   - Toggle state if conditions met

10. **Item updates** (items.cpp):
    - For each item: update animation frame
    - Detect player pickup (zone-based)
    - Apply inventory changes (ammunition, powerups, weapons)

11. **Scenery updates** (scenery.cpp):
    - For each scenery object: update animation frame
    - Apply damage and destruction effects

12. **Weapon updates** (weapons.cpp):
    - For each weapon: read firing state (idle, firing, reloading)
    - Process ammunition tracking and reload timers
    - Trigger projectile creation on fire events
    - Play firing and reload sounds

13. **Polygon trigger processing**:
    - For each polygon entered/exited: activate associated device, monster, item, or platform

14. **Damage and completion checks**:
    - Process accumulated damage events (apply health reduction, death transitions)
    - Check level completion conditions (all monsters defeated, objectives met)
    - Check game-over conditions (player dead, time expired)

15. **Rendering** (RenderMain/RenderOther subsystems):
    - Render visible world geometry and entities to 320x240 framebuffer (software rendering)
    - Update HUD overlay (ammunition, health, oxygen, radar)
    - Swap framebuffer (or update SDL surface)

16. **Audio** (Sound subsystem):
    - Update ambient/environmental sounds (polygon media, platforms)
    - Play triggered sounds (projectiles, effects, items, monsters, devices)
    - Mix and output audio to speaker

17. **Network** (if multiplayer):
    - Transmit local action flags to remote players (ring or star topology)
    - Receive remote action flags and queue for next-tick processing
    - Detect and handle player disconnection (timeout-based)
    - Relay chat messages if any

18. **Lua event dispatch** (if Lua enabled):
    - Invoke registered callbacks for damage, monster death, item pickup, platform switches

19. **Tick counter increment**:
    - Advance global tick counter (used for deterministic RNG, platform timing, animation frame counting)

### Shutdown

1. **Level/game exit**:
   - Call `save_game_file()` to write player state to save slot (if requested)
   - Deallocate all GameWorld entities and world geometry
   - Unload Lua scripts (state serialized to save file)
   - Close all open WAD files (game_wad.cpp)

2. **Preferences and logging**:
   - Call `write_preferences()` to flush user settings to XML/WAD
   - Flush and close log files

3. **Subsystem shutdown**:
   - Network: `NetExit()` closes topology, deallocates thread pools, disconnects from metaserver
   - Audio: Close mixer, deallocate sound resources
   - Rendering: Deallocate shape collections, textures, framebuffer
   - Input: `exit_joystick()`, `exit_mouse()` release device handles
   - Files: Close resource files, unmap memory

4. **Platform cleanup**:
   - CSeries cleanup: Close timer tasks, release GWorlds, unlock resources
   - Engine shutdown: Call `ExitToShell()` (Mac) or `exit()` (SDL)

---

## Data & Control Boundaries

### Ownership and Resource Lifetimes

- **GameWorld state**: Owned by GameWorld subsystem; all entity pools (player, monsters, projectiles, effects, items, scenery) allocated at map load, deallocated at map unload. No inter-frame persistence for effects/projectiles; all state contained in entity arrays indexed by object ID.
- **Action flags**: Generated by Input subsystem each frame; enqueued to Misc::ActionQueues; dequeued by GameWorld main loop. In multiplayer, network system acts as intermediary (enqueue to network buffer, dequeue from network queue each tick).
- **World geometry**: Owned by Files subsystem (loaded from WAD), managed by GameWorld (spatial indices, collision structures). Read-only during gameplay except for platform-driven height/endpoint updates.
- **Preferences**: Owned by Misc::preferences subsystem; global pointers (`graphics_preferences`, `input_preferences`, etc.) read by all subsystems during initialization. Modifications trigger re-initialization of affected subsystems (e.g., changing graphics resolution reloads textures).
- **Rendering resources** (textures, shape collections, fonts): Owned by RenderMain subsystem (not detailed in this batch). Lazily loaded on demand via shape collection handles and unloaded at level exit.
- **Audio resources** (sound definitions, ambient tracks): Owned by Sound subsystem (not detailed in this batch). Loaded at map initialization, played via SoundManager singleton.
- **Network state**: Owned by Network subsystem; ring or star topology, player list, action flag queues, message buffers managed by network topology implementation.
- **File handles**: Owned by Files subsystem; opened via `OpenedFile` RAII wrappers, automatically closed at scope exit.
- **Lua state**: Owned by Lua subsystem; created at script load, destroyed at script close. Per-script global tables persist across frame updates; serialized to save file on level exit.

### Control Flow Boundaries

- **Input ΓåÆ GameWorld**: Action flags are the only input to game world simulation. All control is event-driven via action flag bitsets; no direct GUI-to-world interaction (devices accessed via action flags, not direct API calls).
- **GameWorld ΓåÆ Network**: Player state (position, action flags, damage) flows to network system for distribution. Network system acts as observer; no feedback loop to world (all network updates arrive as action flags or state corrections).
- **GameWorld ΓåÆ Rendering**: Visible world state (player position, entity positions, animation frames, light intensity) read by rendering pipeline each frame. Rendering is read-only; no feedback to world.
- **GameWorld ΓåÆ Audio**: Entity positions and sound event flags read by audio system. Audio playback is asynchronous; no feedback to world.
- **Preferences Γåö All subsystems**: Bidirectional via getter/setter functions. Preference changes trigger subsystem re-initialization (e.g., input sensitivity, graphics resolution).
- **Logging ΓåÆ All subsystems**: Unidirectional; all subsystems log to Misc::Logging. Log level and routing configured via XML; no feedback loop.
- **Error state ΓåÆ All subsystems**: Unidirectional; subsystems set error state on fatal failures. Interface subsystem reads error state and transitions to error-recovery flow.

---

## Notable Risks / Hotspots

### Performance-Critical Code Paths (Dingoo 400 MHz MIPS, 32 MB RAM)

1. **Main loop tick rate** (marathon2.cpp):
   - Target frame rate constrained by monster AI pathfinding (flood_map.cpp) and collision detection. Tight budget; dynamic entity limits calibrated to keep main loop within frame time.
   - **Risk**: Pathfinding cost-evaluation function (`custom_cost_proc` in flood_map.h) may be expensive; prefer clamped polynomial approximations over distance queries.

2. **Monster pathfinding** (flood_map.cpp, pathfinding.cpp):
   - Breadth-first search on polygon adjacency graph for each active monster each frame. Memory allocation proportional to path limit; hash tables for visited set.
   - **Risk**: Number of monsters ├ù graph traversal cost directly proportional to frame time. Dynamic limit prevents runaway; careful cost function essential.

3. **Collision detection** (map.cpp):
   - Line-of-sight queries for monsters/projectiles and player terrain collision. Polygon boundary testing.
   - **Risk**: O(polygon count) per query; spatial indices (BSP or grid) would improve but consume RAM. Current approach assumes small maps.

4. **Fixed-point arithmetic** (world.h, physics.cpp):
   - All coordinates and velocities use 16.16 fixed-point (no floating-point). Deterministic for network play but constrains precision and animation frame granularity.
   - **Risk**: Precision loss on very large or very small scales; cumulative rounding errors over many ticks may diverge on network.

5. **Rendering pipeline** (RenderMain/RenderOther subsystems, not detailed):
   - Software rasterizer only (no OpenGL). Scanline fill, polygon assembly, texture blitting all on CPU.
   - **Risk**: Texture-heavy scenes may cause frame drops. Fixed 320x240 resolution reduces surface area but 1:1 pixel mapping assumes closest integer scaling of artwork.

### Memory Constraints (32 MB Total)

1. **Entity pool allocation**:
   - Monster, projectile, effect, item pools sized via `dynamic_limits.cpp`. Pre-allocation to avoid fragmentation; no per-entity malloc.
   - **Risk**: Exceeding dynamic limit causes new entities to fail silently (no allocation, entity skipped). No error message to player; gameplay may feel unresponsive.

2. **Pathfinding working set**:
   - Flood-fill graph search allocates visited set and queue proportional to path limit. Large maps or many simultaneous searches consume RAM quickly.
   - **Risk**: OOM on large maps; path limit must be tuned per map or dynamically reduced on low-memory condition.

3. **Network packet buffers**:
   - Ring topology requires O(player count) per-tick forwarding; star topology concentrates buffer on hub. Large action queue windows or slow network may cause buffer overflow.
   - **Risk**: Network packet loss causes action flag gaps; network sync algorithm must tolerate missing ticks.

4. **Shape collection and texture caching**:
   - All used shape bitmaps and color tables cached in memory. Lazy-load on first reference; unload at level exit.
   - **Risk**: Shape collections shared across multiple actors (monsters, items, scenery); circular references prevent early unload.

### Determinism and Network Sync

1. **Fixed-point rounding**:
   - All coordinate updates use 16.16 fixed-point arithmetic. Rounding during physics integration may diverge from floating-point simulation over many ticks.
   - **Risk**: Multiplayer desync if any player uses floating-point math; strict fixed-point enforcement essential.

2. **Deterministic RNG**:
   - `world.h` provides `global_random()` and per-entity random seeds. All randomness must use these; system `rand()` forbidden in network play.
   - **Risk**: Calling `std::rand()` or `POSIX rand()` will break network sync; difficult to audit all call sites.

3. **Platform-specific behavior**:
   - Endianness conversion (`byte_swapping.cpp`, `BStream.h`) ensures big-endian wire format. But compiled-in constants (e.g., `#define FIXED_ONE 0x10000`) may differ if compile-time preprocessor state differs across clients.
   - **Risk**: Misaligned preprocessor conditionals between players (e.g., one compiled with `HAVE_LUA`, another without) cause silent desync.

4. **Pathfinding cost function**:
   - Flood-map cost evaluation via custom callback per monster AI. Different cost functions or cost thresholds between clients cause divergent path selection.
   - **Risk**: If cost function uses floating-point or system RNG, network play will desync.

### Dingoo Port-Specific

1. **Shoulder button context sensitivity**:
   - Map view (overhead map toggle) vs. game view (weapon fire) switch on shoulder buttons. Incorrect context tracking causes misfired actions.
   - **Risk**: Race condition if view transition and button press occur in same frame.

2. **320x240 resolution**: 
   - Fixed framebuffer resolution; some UI elements (dialogs, lists) may overflow on smaller dimension. Dialog layout assumes minimum safe area.
   - **Risk**: Wide text strings or many list items cause UI clipping or overflow.

3. **Stripped or reduced subsystems**:
   - Sound (likely stub or minimal), Lua (likely disabled), voice (likely disabled), metaserver (may be disabled), thread priority (stub only).
   - **Risk**: Feature availability divergence between Dingoo build and mainline; players expect feature parity.

4. **ALEPHONE_USERDATA environment variable**:
   - Save directory isolation per episode; if not set, all episodes share same save slots, causing overwrites.
   - **Risk**: User confusion; unset variable silently resets progress.

5. **Cross-compile MIPS toolchain** (dingux-build Makefile):
   - mipsel-linux-g++ compiler; endianness and alignment assumptions may differ from desktop builds.
   - **Risk**: Integer alignment issues on MIPS (stricter than x86); buffer overrun or crash if binary serialization misaligned.
