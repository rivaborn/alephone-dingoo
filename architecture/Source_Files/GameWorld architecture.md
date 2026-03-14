# Subsystem Overview

## Purpose

The GameWorld subsystem implements the complete game world simulation for the Marathon engine running on constrained Dingoo hardware. It manages world geometry (polygon-based representation), all interactive entities (player, monsters, items, projectiles, effects, scenery), physics simulation, environmental systems (platforms, lights, liquids), and the main game loop that coordinates per-frame updates across all subsystems. All data structures and systems are designed for deterministic networked multiplayer, save/load serialization, and XML-driven runtime configuration.

## Key Files

| File | Role |
|------|------|
| **map.cpp / map.h** | Core world geometry and object lifecycle; polygon/line/endpoint/side structures; collision detection; world object creation/removal and per-frame updates |
| **map_constructors.cpp** | Map geometry construction and precalculation (areas, adjacencies, lightsource assignment); serialization to/from binary streams |
| **world.cpp / world.h** | Coordinate system, fixed-point arithmetic, trigonometric lookups, geometric transformations, distance calculations, deterministic random number generation |
| **marathon2.cpp** | Main game loop orchestration: world initialization, per-tick updates, level transitions, action queue processing, damage/completion checks |
| **player.cpp / player.h** | Player entity lifecycle, per-tick state updates, damage/healing, weapon and powerup inventory, oxygen mechanics, death/respawn, network sync |
| **physics.cpp / physics.h / physics_models.h** | Player physics simulation: position/velocity, gravity, jumping, collision resolution, media interaction, animation state, network lag compensation |
| **monsters.cpp / monsters.h / monster_definitions.h** | NPC lifecycle, frame-by-frame AI updates, pathfinding, target acquisition, combat behavior, damage application, animation, serialization |
| **flood_map.cpp / flood_map.h** | Stateful graph search pathfinding using polygon adjacency with breadth-first/best-first/depth-first traversal and custom cost evaluation |
| **pathfinding.cpp** | Pool-based path management; creates paths via flood-fill; advances entities along waypoints |
| **projectiles.cpp / projectiles.h / projectile_definitions.h** | Projectile physics (gravity, wander, guided steering), collision detection, media penetration, damage-on-impact, detonation effects, contrails |
| **effects.cpp / effects.h / effect_definitions.h** | Temporary visual/audio phenomena (explosions, splashes, contrails); animation, sound playback, serialization; ~60+ effect types |
| **platforms.cpp / platforms.h / platform_definitions.h** | Movable platform state machine, movement physics, geometry updates (polygon/endpoint heights), obstruction detection, player/monster interaction, media submersion |
| **lightsource.cpp / lightsource.h** | Dynamic lighting with 6-state machine (becoming/primary/secondary Γåö active/inactive); intensity animation functions (constant, linear, smooth, flicker); serialization |
| **media.cpp / media.h / media_definitions.h** | Liquid media (water, lava, goo, sewage, Jjaro): state updates, height from light intensity, damage, detonation effects, ambient/event sounds |
| **devices.cpp** | Interactive control panels: state initialization, per-tick recharging (shields/oxygen), player interaction detection, toggle conditions, XML overrides |
| **items.cpp / items.h / item_definitions.h** | Item placement, player pickup mechanics, inventory management, animation, zone-based triggering, environment/network restrictions |
| **scenery.cpp / scenery.h / scenery_definitions.h** | Static/destroyable world objects: creation, animation, randomization, damage/destruction effects, XML configuration (61 pre-defined scenery types) |
| **weapons.cpp / weapons.h / weapon_definitions.h** | Weapon state machines (idle/firing/reloading), dual-trigger/dual-wield mechanics, ammunition tracking, shell casing effects, environment compatibility, XML config |
| **dynamic_limits.cpp / dynamic_limits.h** | Configurable runtime limits on entity counts (objects, monsters, projectiles, effects, collision buffers); XML-driven configuration with fallback defaults |
| **editor.h** | Map version constants and geometric bounds for Marathon 1/2/Infinity format compatibility |
| **TickBasedCircularQueue.h** | Tick-indexed circular queue data structure for per-frame action flag storage with lock-free reader/writer access patterns |

## Core Responsibilities

- **World Geometry & Collision**: Create and maintain polygon-based world representation; detect and resolve collisions with terrain, objects, line-of-sight, sound propagation
- **Entity Lifecycle**: Create, update, and destroy players, monsters, items, projectiles, effects, and scenery within the world
- **Physics Simulation**: Implement player physics (movement, gravity, jumping, external forces, media interaction); monster pathfinding physics; projectile ballistics and collision
- **Combat System**: Apply damage to entities, track damage given/taken, manage invincibility/invisibility/powerups, handle death and respawn
- **Environmental Interaction**: Manage movable platforms (doors, elevators, crushers) with geometry updates; implement dynamic lighting with state transitions and intensity animation; handle liquid media with effects, damage, and submersion
- **Inventory & Items**: Manage player weapon/ammunition/powerup inventory; handle item pickup, consumption, and environmental restrictions
- **Interactive Devices**: Process player interaction with control panels (recharging, toggles, terminal access, save points)
- **Main Game Loop**: Orchestrate per-frame updates across all subsystems, process action queues, detect level completion and game-over conditions
- **Serialization**: Pack/unpack all world state for save/load and network transmission; maintain format compatibility across Marathon versions
- **Configuration**: Parse and apply XML-driven customization for game entities, damage definitions, physics models, lighting, media, and weapons; support dynamic entity limits

## Key Interfaces & Data Flow

**Exported to other subsystems:**
- **To Rendering**: `map.h` geometry accessors; shape/animation data via `interface.h`; visibility queries; world object positions/states
- **To Audio**: `SoundManager.h` integration; polygon-based sound propagation; object/projectile/effect sound effects
- **To Input**: `ActionQueues.h` action flag processing; player movement/rotation/weapon control
- **To Network**: Serialized player/entity state for multiplayer sync; action queue buffering for lag compensation
- **To UI/HUD**: Player health/oxygen/ammo/powerup state; game completion flags; damage notifications

**Consumed from other subsystems:**
- **Rendering System** (`interface.h`): Shape collections, animation metadata, transfer modes
- **Audio System** (`SoundManager.h`): Sound playback, resource loading
- **Input System** (`ActionQueues.h`): Real-time and network-deferred action flags
- **Scripting System** (`lua_script.h`): Optional callbacks for lights, platforms, monsters, item pickup (gated by `#ifdef HAVE_LUA`)
- **Network System** (`network.h`, `network_games.h`): Multiplayer mode detection, netdead players, team information, network parameters
- **Configuration System** (`XML_ElementParser.h`): Runtime customization of items, monsters, weapons, platforms, lighting, media, physics, damage

## Runtime Role

**Initialization Phase:**
- Load map and initialize world geometry, collision structures, and spatial indices
- Instantiate player entity with initial state and inventory
- Place initial monsters/items according to map placement frequency data
- Initialize all dynamic systems (lights, platforms, media) with per-map state
- Preload shape collections and sound effects required by visible geometry

**Per-Frame Update Loop (main loop in `marathon2.cpp`):**
1. Process player action queue (deferred from input or network, or predictive from local input)
2. Update player physics (position, velocity, collision, media interaction)
3. Update all monsters (AI, pathfinding, animation, combat, physics)
4. Update all projectiles (movement, collision detection, damage application, detonation)
5. Update all effects (animation, sound, termination detection)
6. Update all platforms (movement, state transitions, geometry updates)
7. Update all lights (phase advancement, state transitions, intensity recalculation)
8. Update media (height from light intensity, movement)
9. Update scenery (animation, randomization)
10. Update items/weapons (animation, firing cycles, reloading)
11. Process polygon triggers (devices, platforms, monsters, items activated by entry)
12. Resolve damage events and apply effects
13. Check for level completion or game-over conditions
14. Render frame using accumulated world state

**Shutdown Phase:**
- Unload map and deallocate all world structures
- Clean up entity lists and dynamic memory
- Save game state to persistent storage if requested

## Notable Implementation Details

- **Polygon-based World Representation**: Uses adjacency graphs of polygons for pathfinding, spatial queries, and audio propagation (no axis-aligned bounding boxes or voxel grids)
- **Fixed-Point Arithmetic**: All world coordinates and physics use 16.16 fixed-point representation (no floating-point) for deterministic network play
- **State Machine Patterns**: Lights, platforms, weapons, and monsters employ explicit state machines with phase counters to ensure deterministic, tick-based transitions
- **Cost-Based Pathfinding**: Flood-fill algorithm supports custom cost callbacks, enabling monsters to avoid lava, prefer open areas, or prefer high ground
- **Dynamic Entity Limits**: Runtime limits on objects, monsters, projectiles, effects configured via XML; pathfinding memory allocated proportionally to path limit
- **Network Lag Compensation**: Predictive local movement simulation decouples client responsiveness from server-authoritative state updates; action queue buffering enables deferred network actions
- **Deterministic Random Number Generation**: Global and per-entity random seeds ensure reproducible behavior across network sync and playback
- **Conditional Scripting**: Lua hooks at key points (light activation, platform state, monster damage) gated by `#ifdef HAVE_LUA` for legacy/optional integration
- **Double-Click Save Detection**: Pattern buffer in devices uses frame-level detection to distinguish intentional network save overwrites
- **Environment-Driven Physics**: Platforms affect monster pathfinding cost; media affects projectile penetration, item placement restrictions, and player oxygen depletion
- **XML-Driven Overrides**: All entity definitions (weapons, monsters, items, effects, platforms, media) can be customized via MML/XML configuration without code changes
