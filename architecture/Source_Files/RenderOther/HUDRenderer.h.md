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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `weapon_interface_ammo_data` | struct | Per-trigger ammo display config (grid layout, bullet graphics, direction) |
| `weapon_interface_data` | struct | Complete weapon panel visual definition (item ID, textures, name placement, ammo data) |
| `interface_state_data` | struct | Dirty flags tracking which HUD components need redraw (ammo, weapon, shield, oxygen) |
| `HUD_Class` | class | Abstract base class defining HUD rendering interface and state management |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `interface_state` | `interface_state_data` | global | Tracks which HUD components are dirty and need redrawing |
| `weapon_interface_definitions` | `weapon_interface_data[10]` | global | Visual/layout templates for each of 10 weapon panel configurations |

## Key Functions / Methods

### update_everything
- Signature: `bool update_everything(short time_elapsed)`
- Purpose: Main entry point for per-frame HUD updates; orchestrates all state changes
- Inputs: `time_elapsed` (ticks since last frame)
- Outputs/Return: bool (true if anything changed)
- Side effects: Updates internal dirty flags, calls all protected update methods

### draw_all_entity_blips
- Signature: `virtual void draw_all_entity_blips(void)`
- Purpose: Render all motion sensor entity blips (friends, aliens, enemies) for current frame
- Side effects: Calls `draw_entity_blip()` for each visible entity

### motion_sensor_scan
- Signature: `void motion_sensor_scan(short ticks_elapsed)`
- Purpose: Scan world for entities and update motion sensor state
- Side effects: Updates motion sensor internal state; calls virtual `update_motion_sensor()`

### Virtual Rendering Primitives
- `DrawShape()`, `DrawShapeAtXY()` ΓÇô render textured/shaped sprites with optional transparency
- `DrawText()` ΓÇô render text in specified rectangle with formatting flags
- `FillRect()`, `FrameRect()` ΓÇô fill or outline rectangles with solid color
- `DrawTexture()` ΓÇô specialized 3D texture rendering for motion sensor
- `SetClipPlane()` / `DisableClipPlane()` ΓÇô circular clipping for motion sensor viewport
- `draw_or_erase_unclipped_shape()` ΓÇô helper for entity blip rendering (pure virtual)

## Control Flow Notes
Fits into frame/update pipeline: `update_everything()` is called once per game tick. It serially invokes protected update methods (`update_suit_energy()`, `update_weapon_panel()`, etc.), which check dirty flags and call virtual rendering primitives. Motion sensor has its own update/render split (`update_motion_sensor()` updates state; `render_motion_sensor()` draws it).

## External Dependencies
- **Player state:** accesses `player.h` (suit energy, oxygen, weapons, inventory)
- **Weapons & Items:** `weapons.h`, `items.h` for item definitions and ammo counts
- **World:** `map.h`, `world.h` for motion sensor entity locations
- **Sound:** `SoundManager.h` for button click feedback
- **Drawing:** `screen_drawing.h` for screen coordinates and rectangles
- **Network:** `network_games.h` for multiplayer-specific HUD elements
- **Texture IDs:** shape descriptors (panel art, bars, motion sensor mounts/icons) defined in enums at file top

**Notes:** Texture enum values (_energy_bar, _magnum_panel, etc.) are hardcoded indices into a resource system loaded elsewhere. The class is designed as an abstract interface to support multiple rendering backends without duplicating high-level HUD logic.
